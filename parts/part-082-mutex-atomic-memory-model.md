# Part 82: std::mutex, std::atomic, C++ Memory Model (Step 649–656)

> Module G — Concurrency และ Performance Engineering | Part 82 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 649–656
> Part ก่อนหน้า: [Part 81 — std::thread และ Concurrency ใน Modern C++](./part-081-std-thread.md) | Part ถัดไป: [Part 83 — std::future, std::async, std::promise](./part-083-async-future-promise.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ใช้ `std::mutex` คู่กับ `std::lock_guard` และ `std::unique_lock` แก้ Race Condition
   ในสไตล์ Modern C++ ที่ผูก RAII (Part 68) เข้ากับการล็อก/ปลดล็อกโดยอัตโนมัติ
2. เลือกใช้ `std::recursive_mutex` และ `std::timed_mutex` ได้อย่างถูกต้องตามสถานการณ์
   พร้อมเทียบกับ `PTHREAD_MUTEX_RECURSIVE` ที่เรียนไปแล้วใน Part 32
3. อธิบายได้ว่า `std::atomic<T>` คืออะไร ทำงานอย่างไรในระดับ hardware และทำไมมันถึง
   เร็วกว่า mutex สำหรับข้อมูลพื้นฐาน พร้อมพิสูจน์ด้วยการวัดความเร็วจริง
4. อธิบายแนวคิดพื้นฐานของ **C++ Memory Model** และความหมายของ `memory_order` แต่ละแบบ
   (`relaxed`, `acquire`/`release`, `seq_cst`) ในระดับที่นำไปใช้ได้จริง
5. ใช้ `std::lock()` และ `std::scoped_lock` (C++17) ล็อกหลาย mutex พร้อมกันได้อย่าง
   ปลอดภัยจาก Deadlock โดยไม่ต้องกำหนด Lock Ordering เองด้วยมือเหมือนที่ Part 32 สอน
6. เลือกได้อย่างเหมาะสมว่าเมื่อไหร่ควรใช้ mutex และเมื่อไหร่ควรใช้ atomic ในการออกแบบ
   โปรแกรม concurrent จริง

---

## 82.1 ทบทวน RAII กับ Mutex: `std::mutex`, `std::lock_guard`, `std::unique_lock` (Step 649)

ใน Part 32 เราแก้ Race Condition ด้วย `pthread_mutex_t` ผ่านรูปแบบ `lock()` /
`unlock()` ที่ต้องเรียกคู่กันด้วยมือทุกครั้ง — และเห็นบั๊กคลาสสิกจากการ "ลืม unlock ตอน
early return" ใน 32.4 ไปแล้ว

Modern C++ แก้ปัญหานี้ด้วยการนำหลักการ **RAII** ที่เรียนใน Part 68 มาผูกกับ mutex
โดยตรง: `std::mutex` คือตัว mutex ดิบๆ (คล้าย `pthread_mutex_t`) แต่แทนที่จะเรียก
`.lock()`/`.unlock()` เองตรงๆ เราใช้ **`std::lock_guard`** ห่อมันไว้อีกชั้น —
`lock_guard` จะ `.lock()` ให้ตอนสร้าง (constructor) และ `.unlock()` ให้อัตโนมัติตอน
destruct (ออกจาก scope) เสมอ ไม่ว่าจะออกทางไหนก็ตาม แม้แต่ผ่าน exception

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

constexpr int NUM_THREADS = 4;
constexpr long INCREMENTS_PER_THREAD = 1'000'000;

long counter = 0;
std::mutex counter_mutex;

void increment_counter() {
    for (long i = 0; i < INCREMENTS_PER_THREAD; ++i) {
        std::lock_guard<std::mutex> lock(counter_mutex); // ล็อกตอนสร้าง, ปลดล็อกอัตโนมัติตอนจบ scope
        ++counter;
    } // lock ถูกปลดโดย destructor ของ lock_guard ตรงนี้ ทุกครั้งที่ออกจาก loop body
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < NUM_THREADS; ++i) {
        threads.emplace_back(increment_counter);
    }
    for (auto& t : threads) {
        t.join();
    }

    long expected = static_cast<long>(NUM_THREADS) * INCREMENTS_PER_THREAD;
    std::cout << "Expected: " << expected << "\n";
    std::cout << "Actual:   " << counter << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 mutex_lock_guard.cpp -o mutex_lock_guard
./mutex_lock_guard
```

ผลลัพธ์ (ถูกต้องทุกครั้งที่รัน เหมือนกับ `mutex_fix.c` ใน Part 32):

```
Expected: 4000000
Actual:   4000000
```

เทียบกับ pthread ที่ต้องเขียน:

```c
pthread_mutex_lock(&counter_mutex);
counter++;
pthread_mutex_unlock(&counter_mutex);
```

`std::lock_guard<std::mutex> lock(counter_mutex);` บรรทัดเดียวทำหน้าที่แทนทั้งคู่ —
**และรับประกันว่าจะ unlock เสมอ** แม้จะมี early `return` หรือ exception เกิดขึ้นระหว่าง
ทาง (แก้ปัญหา "ลืม unlock ตอน early return" จาก Part 32 หัวข้อ 32.4 ได้อย่างสมบูรณ์
โดยที่โปรแกรมเมอร์ไม่ต้องคิดเรื่องนี้เองเลยแม้แต่น้อย)

### `std::unique_lock`: ยืดหยุ่นกว่า `lock_guard`

`std::lock_guard` เรียบง่ายและมี overhead ต่ำที่สุด แต่ **ล็อกครั้งเดียวตอนสร้าง แล้ว
ปลดล็อกครั้งเดียวตอน destruct เท่านั้น** ไม่มีความยืดหยุ่นอื่นใด ถ้าต้องการปลดล็อกชั่วคราว
กลางทาง (เช่น เพื่อทำงานที่ไม่ต้องการ critical section แทรกอยู่ตรงกลาง) ต้องใช้
**`std::unique_lock`** แทน:

```cpp
#include <iostream>
#include <mutex>
#include <thread>

std::mutex log_mutex;
int shared_balance = 1000;

void withdraw(int amount) {
    // std::unique_lock ยืดหยุ่นกว่า lock_guard: ปลดล็อกชั่วคราวได้กลางทาง (unlock()/lock())
    std::unique_lock<std::mutex> lock(log_mutex);

    if (amount > shared_balance) {
        std::cout << "[withdraw] เงินไม่พอ ยกเลิกการถอน " << amount << "\n";
        return; // unique_lock destructor จะปลดล็อกให้อัตโนมัติ แม้ return กลางฟังก์ชัน
    }

    shared_balance -= amount;
    std::cout << "[withdraw] ถอน " << amount << " สำเร็จ เหลือ " << shared_balance << "\n";

    // ตัวอย่างการปลดล็อกชั่วคราวเพื่อทำงานที่ไม่ต้องการ critical section
    // (เช่น เขียน log ลงไฟล์ที่ใช้เวลานาน) แล้วค่อย lock กลับ
    lock.unlock();
    std::cout << "[withdraw] (จำลอง) กำลังเขียน log แบบไม่ถือ mutex อยู่...\n";
    lock.lock();

    std::cout << "[withdraw] จบการทำธุรกรรม\n";
}

int main() {
    std::thread t1(withdraw, 300);
    std::thread t2(withdraw, 900);
    t1.join();
    t2.join();
    std::cout << "[main] ยอดคงเหลือสุดท้าย: " << shared_balance << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread unique_lock_demo.cpp -o unique_lock_demo
./unique_lock_demo
```

ผลลัพธ์ (ลำดับอาจสลับกันไปมาระหว่างการรันแต่ละครั้ง เพราะขึ้นกับ scheduler):

```
[withdraw] ถอน 300 สำเร็จ เหลือ 700
[withdraw] (จำลอง) กำลังเขียน log แบบไม่ถือ mutex อยู่...
[withdraw] จบการทำธุรกรรม
[withdraw] เงินไม่พอ ยกเลิกการถอน 900
[main] ยอดคงเหลือสุดท้าย: 700
```

### `lock_guard` vs `unique_lock`: เลือกใช้เมื่อไหร่

| หัวข้อ | `std::lock_guard` | `std::unique_lock` |
|---|---|---|
| Overhead | ต่ำที่สุด (แค่ boolean ธรรมดาไม่มีด้วยซ้ำในบาง implementation) | สูงกว่าเล็กน้อย (ต้องเก็บสถานะว่ากำลังถือ lock อยู่หรือไม่) |
| ปลดล็อกชั่วคราวกลางทางได้ไหม | ไม่ได้ | ได้ (`.unlock()`/`.lock()`) |
| ใช้กับ `std::condition_variable` ได้ไหม | ไม่ได้ (Part นี้ยังไม่ใช้ condition_variable แต่จะเกี่ยวข้องถ้าค้นคว้าต่อ) | ได้ — `condition_variable::wait()` ต้องการ `unique_lock` เท่านั้น |
| ใช้กับ `std::lock()`/`std::scoped_lock` แบบ adopt ได้ไหม | ได้ (ผ่าน `std::adopt_lock`) | ได้เช่นกัน |
| แนะนำเมื่อ | ต้องการล็อกแบบง่ายที่สุด ล็อกตลอด scope โดยไม่ปลดกลางทาง (ใช้บ่อยที่สุดในโค้ดทั่วไป) | ต้องการความยืดหยุ่นเพิ่มเติม เช่น ปลดล็อกชั่วคราว, ล็อกแบบ deferred, หรือ move ownership ของ lock ไปมา |

> **หลักปฏิบัติ**: ใช้ `std::lock_guard` เป็นค่า default เสมอ เปลี่ยนไปใช้
> `std::unique_lock` เฉพาะเมื่อต้องการความยืดหยุ่นที่ `lock_guard` ให้ไม่ได้จริงๆ
> เท่านั้น (Core Guidelines หลักการเดียวกับ Part 80: **ใช้เครื่องมือที่เรียบง่ายที่สุด
> เท่าที่ยังตอบโจทย์**)

---

## 82.2 `std::recursive_mutex` และ `std::timed_mutex` (Step 650)

### `std::recursive_mutex`: ล็อกซ้ำได้ในเธรดเดียวกัน

เทียบเท่ากับ `PTHREAD_MUTEX_RECURSIVE` ที่เรียนใน Part 32 หัวข้อ 32.4 ทุกประการ —
ปกติแล้ว `std::mutex` ธรรมดา ถ้า thread เดียวกัน `.lock()` ซ้ำสองครั้งจะเกิด
**Deadlock กับตัวเอง** ทันที (Undefined Behavior ตามมาตรฐาน แต่ implementation ส่วนใหญ่
จะทำให้ thread บล็อกตัวเองค้างตลอดไป) `std::recursive_mutex` แก้ปัญหานี้โดยอนุญาตให้
thread เดียวกัน lock ซ้ำได้หลายชั้น (ต้อง unlock ให้ครบเท่าจำนวนครั้งที่ lock ด้วย):

```cpp
#include <iostream>
#include <mutex>

std::recursive_mutex rec_mutex;

void print_countdown(int n) {
    std::lock_guard<std::recursive_mutex> lock(rec_mutex); // ล็อกซ้ำได้เพราะเป็น thread เดียวกัน
    if (n <= 0) {
        std::cout << "[countdown] จบแล้ว!\n";
        return;
    }
    std::cout << "[countdown] " << n << "\n";
    print_countdown(n - 1); // เรียกซ้ำ -> ล็อก rec_mutex ซ้ำในเธรดเดียวกัน (ชั้นที่ 2, 3, ...)
}

int main() {
    print_countdown(3);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread recursive_mutex_demo.cpp -o recursive_mutex_demo
./recursive_mutex_demo
```

ผลลัพธ์:

```
[countdown] 3
[countdown] 2
[countdown] 1
[countdown] จบแล้ว!
```

ถ้าลองเปลี่ยน `rec_mutex` เป็น `std::mutex` ธรรมดาในโค้ดนี้ โปรแกรมจะ**ค้างตลอดไป**
ตั้งแต่การเรียกซ้ำครั้งแรก (`print_countdown(2)` พยายาม lock mutex ที่ตัวเองถืออยู่แล้ว
จาก `print_countdown(3)`) — เหมือนกับที่ Part 32 อธิบายพฤติกรรมของ
`PTHREAD_MUTEX_NORMAL` ทุกประการ

> **ข้อควรระวัง** (เหมือนที่ Part 32 เตือนไว้กับ pthread): `std::recursive_mutex` มัก
> เป็นสัญญาณว่าการออกแบบโค้ดมีปัญหาเชิงโครงสร้าง (ฟังก์ชันที่ถือ lock อยู่แล้วเรียก
> ฟังก์ชันอื่นที่ต้องการ lock เดียวกันอีกที) การใช้งานจริงควรพิจารณาปรับโครงสร้างโค้ด
> ให้ไม่ต้อง lock ซ้ำเสมอถ้าเป็นไปได้ ก่อนจะหันมาพึ่ง `recursive_mutex`

### `std::timed_mutex`: ล็อกแบบมีกำหนดเวลา

`std::timed_mutex` เพิ่มความสามารถที่ `std::mutex` ธรรมดาไม่มี: **ลองล็อกโดยกำหนดเวลา
รอสูงสุด** ผ่าน `.try_lock_for(duration)` หรือ `.try_lock_until(time_point)` แทนที่จะ
บล็อกรอตลอดไปเหมือน `.lock()` ปกติ:

```cpp
#include <chrono>
#include <iostream>
#include <mutex>
#include <thread>

std::timed_mutex resource_mutex;

void long_task() {
    std::lock_guard<std::timed_mutex> lock(resource_mutex);
    std::cout << "[long_task] ถือ lock อยู่ กำลังทำงานหนัก 300ms\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(300));
    std::cout << "[long_task] ทำงานเสร็จ ปล่อย lock\n";
}

void impatient_task() {
    std::this_thread::sleep_for(std::chrono::milliseconds(50)); // ให้ long_task ล็อกก่อน
    std::cout << "[impatient_task] ลองขอ lock โดยรอไม่เกิน 100ms...\n";
    if (resource_mutex.try_lock_for(std::chrono::milliseconds(100))) {
        std::cout << "[impatient_task] ได้ lock! (ไม่ควรเกิดขึ้นในตัวอย่างนี้)\n";
        resource_mutex.unlock();
    } else {
        std::cout << "[impatient_task] รอเกิน 100ms แล้วไม่ได้ lock เลยยกเลิกไปทำอย่างอื่นแทน\n";
    }
}

int main() {
    std::thread t1(long_task);
    std::thread t2(impatient_task);
    t1.join();
    t2.join();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread timed_mutex_demo.cpp -o timed_mutex_demo
./timed_mutex_demo
```

ผลลัพธ์:

```
[long_task] ถือ lock อยู่ กำลังทำงานหนัก 300ms
[impatient_task] ลองขอ lock โดยรอไม่เกิน 100ms...
[impatient_task] รอเกิน 100ms แล้วไม่ได้ lock เลยยกเลิกไปทำอย่างอื่นแทน
[long_task] ทำงานเสร็จ ปล่อย lock
```

`std::timed_mutex` มีประโยชน์มากในระบบที่ต้อง**ไม่บล็อกค้างนาน** เช่น UI thread ที่ต้อง
ตอบสนองผู้ใช้ตลอดเวลา หรือระบบ real-time ที่มี deadline — ถ้าล็อกไม่สำเร็จภายในเวลาที่
กำหนด โปรแกรมสามารถเลือก "ยกเลิก" หรือ "ลองใหม่" หรือ "แจ้ง error" แทนการรอเฉยๆ ได้
(หลักการเดียวกับ `pthread_mutex_trylock` + backoff ที่ Part 32 หัวข้อ 32.8 แนะนำ
สำหรับป้องกัน Deadlock)

---

## 82.3 `std::atomic<T>`: Lock-Free Operation ระดับ Hardware (Step 651)

### ปัญหาของ Mutex: Overhead จากการล็อก/ปลดล็อก

`std::mutex` แก้ Race Condition ได้อย่างถูกต้อง แต่มี **ต้นทุน (overhead)** ที่ต้องจ่าย
ทุกครั้งที่ล็อก/ปลดล็อก — แม้ในกรณีที่ไม่มี thread อื่นแย่งชิงเลย (`uncontended lock`)
ก็ยังต้องผ่านกลไกภายในของ OS/runtime อยู่ดี และในกรณีที่มีการแย่งชิงจริง (`contended
lock`) thread ที่แพ้จะถูก**บล็อก** (ให้ OS scheduler สลับไปทำงานอื่นแล้วค่อยกลับมาปลุก
ทีหลัง) ซึ่งมี cost สูงกว่าการรอเฉยๆ มาก

สำหรับข้อมูลพื้นฐาน (`int`, `long`, `bool`, pointer) ที่การดำเนินการมีขนาดเล็กมาก
(เช่น การเพิ่มค่าทีละ 1) CPU สมัยใหม่มี **instruction พิเศษระดับ hardware** ที่ทำ
read-modify-write ให้เสร็จในขั้นตอนเดียวแบบ **atomic** (แบ่งแยกไม่ได้ — ไม่มีทางที่
thread อื่นจะเข้ามา "แทรก" ระหว่างขั้นตอนได้เลย) โดยไม่ต้องพึ่งกลไก lock ของ OS แม้แต่
น้อย เช่น `LOCK XADD` บน x86 หรือ `LDXR`/`STXR` บน ARM

`std::atomic<T>` คือ wrapper ของ Standard Library ที่ **ห่อ instruction ระดับ
hardware เหล่านี้** ให้ใช้งานง่ายผ่านภาษา C++ โดยตรง โดยไม่ต้องเขียน inline assembly
เองเลย:

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

constexpr int NUM_THREADS = 4;
constexpr long INCREMENTS_PER_THREAD = 1'000'000;

std::atomic<long> counter{0};

void increment_counter() {
    for (long i = 0; i < INCREMENTS_PER_THREAD; ++i) {
        ++counter; // atomic read-modify-write ระดับ CPU instruction เดียว ไม่ต้องล็อก
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < NUM_THREADS; ++i) {
        threads.emplace_back(increment_counter);
    }
    for (auto& t : threads) {
        t.join();
    }

    long expected = static_cast<long>(NUM_THREADS) * INCREMENTS_PER_THREAD;
    std::cout << "Expected: " << expected << "\n";
    std::cout << "Actual:   " << counter.load() << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 atomic_counter.cpp -o atomic_counter
./atomic_counter
```

ผลลัพธ์ (ถูกต้องทุกครั้งเช่นเดียวกับเวอร์ชัน mutex):

```
Expected: 4000000
Actual:   4000000
```

สังเกตว่าโค้ดนี้ **ไม่มี mutex เลยแม้แต่ตัวเดียว** — `++counter` ที่เป็น `std::atomic<long>`
คือการดำเนินการแบบ atomic ในตัวเอง ไม่มีการล็อกใดๆ (ในความหมายของ OS-level lock) เกิดขึ้น
เลย แต่ยังคงปลอดภัยจาก Race Condition 100% เพราะ CPU รับประกันความถูกต้องของ
read-modify-write นี้ในระดับ hardware โดยตรง

### Mutex vs Atomic: ตารางเปรียบเทียบ

| หัวข้อ | `std::mutex` + `lock_guard` | `std::atomic<T>` |
|---|---|---|
| กลไกเบื้องหลัง | OS-level lock (futex บน Linux) — thread ที่แพ้ถูกบล็อกจริง | CPU instruction ระดับ hardware (เช่น `LOCK XADD`) — ไม่มีการบล็อก thread |
| ใช้ป้องกันอะไรได้ | Critical Section **ขนาดใดก็ได้** (หลายบรรทัด, หลายตัวแปร, เรียกฟังก์ชันอื่นได้) | เฉพาะการดำเนินการ**เดียว**บนตัวแปรชนิดพื้นฐาน (`int`, `bool`, pointer ฯลฯ) ต่อครั้งเท่านั้น |
| ความเร็ว (uncontended) | ช้ากว่า | เร็วกว่ามาก |
| ความเร็ว (contended สูง) | ช้าลงมาก (thread ต้อง sleep/wake) | ยังคงเร็วกว่า แต่ CPU cache อาจ "สั่นไหว" ระหว่าง core ได้ถ้าแย่งกันเขียนตัวแปรเดียวกันถี่มาก |
| ความซับซ้อนในการใช้ให้ถูก | ตรงไปตรงมา (lock ครอบ critical section) | ต้องเข้าใจ `memory_order` เพื่อใช้ให้ถูกต้องในกรณีซับซ้อน (82.5) |
| เหมาะกับ | Critical Section ที่ซับซ้อน, ต้องแก้หลายตัวแปรพร้อมกันแบบ atomic ร่วมกัน | ตัวนับ (counter), flag (`bool`), สถานะง่ายๆ ที่เป็นค่าเดียวโดดๆ |

### วัดความเร็วจริง: Atomic เร็วกว่า Mutex แค่ไหน

รันทั้งสองเวอร์ชันด้วยจำนวนการเพิ่มค่าเท่ากันทุกประการ (4 thread × 1,000,000 ครั้ง)
แล้ววัดเวลาด้วย `time`:

```bash
time ./mutex_lock_guard
time ./atomic_counter
```

ผลลัพธ์ตัวอย่างจริงที่วัดได้ (ตัวเลขจะต่างกันไปตามสเปกเครื่อง แต่แนวโน้มเหมือนกันเสมอ):

| เวอร์ชัน | เวลาที่ใช้จริง (real) |
|---|---|
| `std::mutex` + `lock_guard` | ~0.25 วินาที |
| `std::atomic<long>` | ~0.07 วินาที |

**Atomic เร็วกว่า mutex ประมาณ 3-4 เท่า** ในกรณีนี้ (ตัวเลขจริงขึ้นกับจำนวน core, สถาปัตยกรรม
CPU, และระดับการแย่งชิงกัน) เหตุผลหลักคือ mutex version ทุกครั้งที่ increment ต้องผ่าน
ขั้นตอน lock/unlock ที่มี overhead จากการเรียก system call เมื่อเกิดการบล็อกจริง (ผ่าน
`futex` เหมือนที่ Part 32 อธิบายไว้) ในขณะที่ atomic version ทำทุกอย่างจบภายใน CPU
instruction เดียวโดยไม่ต้องออกไปยัง kernel เลย

> **ข้อควรระวัง**: ตัวเลขนี้**ไม่ได้แปลว่า atomic ดีกว่า mutex เสมอไป** — atomic ใช้ได้
> เฉพาะเมื่อ critical section คือ "การดำเนินการเดียว" บนตัวแปรเดียว ถ้าต้องแก้ไขหลาย
> ตัวแปรให้สอดคล้องกัน (เช่น ย้ายเงินระหว่าง 2 บัญชีที่ต้องลดบัญชีหนึ่งพร้อมเพิ่มอีกบัญชี
> แบบ atomic ร่วมกัน) ต้องใช้ mutex เท่านั้น เพราะ atomic ไม่มีแนวคิดเรื่อง "critical
> section ที่ครอบหลายคำสั่ง" ได้เลย

---

## 82.4 C++ Memory Model เบื้องต้น: `memory_order` (Step 652–653)

หัวข้อนี้เป็นเรื่องที่ลึกที่สุดของ Part นี้ — เราจะอธิบายให้เข้าใจได้ในระดับที่ **นำไปใช้
งานจริงได้** โดยไม่ลงลึกทฤษฎีมากเกินความจำเป็น

### ปัญหาที่ Memory Model แก้: การเรียงลำดับคำสั่งที่ Compiler/CPU อาจ "สลับ" ได้

Compiler และ CPU สมัยใหม่ทำ **Optimization** หลายอย่างที่อาจ **สลับลำดับการทำงานจริง**
ของคำสั่งที่ไม่ได้ขึ้นต่อกันโดยตรง (ตราบใดที่ผลลัพธ์สุดท้ายของ**thread เดียว**ยังถูกต้อง
เหมือนเดิม) เพื่อให้โปรแกรมทำงานเร็วขึ้น ปัญหาคือ: **การสลับลำดับที่ "ปลอดภัย" สำหรับ
thread เดียว อาจไม่ปลอดภัยเลยเมื่อมี thread อื่นสังเกตเห็นผลลัพธ์ระหว่างทาง**

`std::atomic` operation แต่ละตัวรับ parameter เสริมชื่อ `std::memory_order` ที่บอกว่า
**"อนุญาตให้สลับลำดับกับคำสั่งข้างเคียงได้มากแค่ไหน"** ยิ่งอนุญาตให้สลับได้มาก
(อ่อนกว่า) ก็ยิ่งเร็ว แต่ต้องเข้าใจผลกระทบให้ถูกต้อง

### `memory_order_relaxed`: เร็วที่สุด รับประกันแค่ Atomicity

`memory_order_relaxed` รับประกัน**แค่ว่าตัวการดำเนินการเองเป็น atomic** (ไม่มีใครเห็น
ค่ากึ่งกลางระหว่างการอ่าน-แก้-เขียน) แต่**ไม่รับประกันลำดับการมองเห็นกับตัวแปรอื่นข้างๆ
เลย** เหมาะที่สุดกับ **ตัวนับล้วนๆ ที่ไม่ต้อง synchronize กับข้อมูลอื่น**:

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

constexpr int NUM_THREADS = 4;
constexpr long INCREMENTS_PER_THREAD = 1'000'000;

std::atomic<long> counter{0};

void increment_relaxed() {
    for (long i = 0; i < INCREMENTS_PER_THREAD; ++i) {
        // memory_order_relaxed: รับประกันแค่ว่าตัว increment เอง atomic (ไม่มีใครเห็นค่ากึ่งกลาง)
        // แต่ไม่รับประกันลำดับการมองเห็นกับตัวแปรอื่นๆ ข้าง เหมาะกับ "ตัวนับล้วนๆ"
        // ที่ไม่ต้อง synchronize กับข้อมูลอื่นเลย
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < NUM_THREADS; ++i) {
        threads.emplace_back(increment_relaxed);
    }
    for (auto& t : threads) {
        t.join();
    }

    std::cout << "Expected: " << static_cast<long>(NUM_THREADS) * INCREMENTS_PER_THREAD << "\n";
    std::cout << "Actual:   " << counter.load(std::memory_order_relaxed) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 memory_order_relaxed.cpp -o memory_order_relaxed
./memory_order_relaxed
```

ผลลัพธ์:

```
Expected: 4000000
Actual:   4000000
```

**ทำไมยังถูกต้อง 100% ทั้งที่ "relaxed" ฟังดูไม่เข้มงวด?** เพราะ `fetch_add` เป็น
**read-modify-write แบบ atomic เดี่ยวๆ** อยู่แล้วไม่ว่าจะใช้ memory order แบบไหนก็ตาม
— ตัว "ผลรวมสุดท้าย" ของการเพิ่มค่าจะถูกต้องเสมอ สิ่งที่ `relaxed` **ไม่รับประกัน** คือ
ลำดับที่ thread อื่นจะ "เห็น" การเปลี่ยนแปลงของตัวแปรอื่นๆ ที่ไม่เกี่ยวกับ `counter`
เทียบกับการเปลี่ยนแปลงของ `counter` เอง — ถ้าโปรแกรมมีแค่ตัวนับตัวเดียวไม่มีข้อมูลอื่น
ต้อง synchronize ด้วย `relaxed` จึงปลอดภัยและเร็วที่สุด

### `memory_order_acquire`/`release`: Synchronize ข้อมูลระหว่าง Thread

สถานการณ์ที่ซับซ้อนกว่าคือเมื่อ thread หนึ่งเตรียมข้อมูล (ที่ไม่ใช่ atomic) แล้วต้อง
"ส่งสัญญาณ" ให้อีก thread รู้ว่าข้อมูลพร้อมแล้ว — กรณีนี้ต้องมั่นใจว่า **thread ที่รับ
สัญญาณจะเห็นข้อมูลที่เตรียมไว้ครบถ้วนจริงๆ ไม่ใช่เห็นแค่บางส่วน**

- **`memory_order_release`** (ใช้ตอน**เขียน**): รับประกันว่าการเขียนข้อมูล**ทุกอย่างที่
  อยู่ก่อนหน้า**ในโค้ดของ thread เดียวกันจะ "เสร็จสมบูรณ์และมองเห็นได้" ก่อนที่การเขียน
  แบบ release ครั้งนี้จะเกิดขึ้น
- **`memory_order_acquire`** (ใช้ตอน**อ่าน**): รับประกันว่าการอ่าน**ทุกอย่างที่ตามมา**
  หลังจากนี้ในโค้ดของ thread เดียวกัน จะเห็นข้อมูลที่ถูกเขียนไว้ก่อนการ release ที่จับคู่
  กันสำเร็จแล้วครบถ้วน

เมื่อ acquire (ฝั่งอ่าน) "จับคู่" กับ release (ฝั่งเขียน) บนตัวแปร atomic ตัวเดียวกันได้
สำเร็จ จะเกิดสิ่งที่เรียกว่า **happens-before relationship** — รับประกันว่าทุกอย่างที่
เขียนไว้ก่อน release จะมองเห็นได้แน่นอนหลัง acquire:

```cpp
#include <atomic>
#include <iostream>
#include <string>
#include <thread>

std::string payload; // ตัวแปรธรรมดา ไม่ใช่ atomic
std::atomic<bool> data_ready{false};

void producer() {
    payload = "ข้อมูลสำคัญจาก producer"; // (1) เขียนข้อมูลธรรมดาก่อน
    data_ready.store(true, std::memory_order_release); // (2) "ปล่อยสัญญาณ" ว่าข้อมูลพร้อมแล้ว
    // memory_order_release รับประกันว่าการเขียนทุกอย่างก่อนหน้านี้ (รวม payload)
    // จะ "มองเห็นได้" โดย thread อื่นที่ทำ acquire บนตัวแปรเดียวกันสำเร็จ
}

void consumer() {
    // วนรอจนกว่า data_ready จะเป็น true ด้วย memory_order_acquire
    while (!data_ready.load(std::memory_order_acquire)) {
        // busy-wait สั้นๆ เพื่อความง่ายของตัวอย่าง (งานจริงควรใช้ condition_variable)
    }
    // เมื่อมาถึงจุดนี้ได้ รับประกันว่า payload ถูกเขียนเสร็จสมบูรณ์แล้วแน่นอน
    // เพราะ acquire "จับคู่" กับ release ของ producer (happens-before relationship)
    std::cout << "[consumer] อ่านค่า payload ได้อย่างปลอดภัย: " << payload << "\n";
}

int main() {
    std::thread p(producer);
    std::thread c(consumer);
    p.join();
    c.join();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 acquire_release.cpp -o acquire_release
./acquire_release
```

ผลลัพธ์ (ถูกต้องทุกครั้งที่รันซ้ำ):

```
[consumer] อ่านค่า payload ได้อย่างปลอดภัย: ข้อมูลสำคัญจาก producer
```

ประเด็นสำคัญที่สุดคือ **`payload` เป็น `std::string` ธรรมดา ไม่ใช่ atomic เลย** —
ถ้าไม่มี acquire/release กำกับ compiler/CPU อาจ**สลับลำดับ**ให้ `data_ready.store(true)`
มองเห็นได้ก่อนที่ `payload` จะถูกเขียนเสร็จจริงๆ ทำให้ consumer อ่าน `payload` ที่ยัง
เขียนไม่เสร็จ (Race Condition ที่ตรวจจับยากมาก เพราะเกิดขึ้นเฉพาะบาง timing เท่านั้น)
acquire/release แก้ปัญหานี้โดยสร้าง "จุดซิงค์" ระหว่างสอง thread ผ่านตัวแปร atomic
เพียงตัวเดียว (`data_ready`) โดยไม่ต้องทำให้ `payload` เป็น atomic ไปด้วย

### `memory_order_seq_cst`: ค่า Default ที่เข้มงวดที่สุด

ถ้าไม่ระบุ `memory_order` เลย (เช่น `counter.fetch_add(1)` หรือ `++counter` ตรงๆ)
ค่า default คือ **`memory_order_seq_cst`** (Sequentially Consistent) ซึ่งเข้มงวดที่สุด
— รับประกันว่า**ทุก thread มองเห็นลำดับการดำเนินการ atomic ทั้งหมดในโปรแกรมตรงกันทุก
ประการ** เหมือนกับว่าโปรแกรมรันแบบ interleaved (สลับกันทีละคำสั่ง) บน thread เดียว
เป็นโหมดที่ **เข้าใจง่ายที่สุดและปลอดภัยที่สุด** แต่ก็มี**overhead สูงที่สุด**ด้วยเช่นกัน
(ต้องใส่ memory barrier ที่แพงกว่าในระดับ hardware)

### ตารางสรุป `memory_order` ที่ใช้บ่อยที่สุด

| memory_order | รับประกันอะไร | เร็วแค่ไหน | ใช้เมื่อ |
|---|---|---|---|
| `memory_order_relaxed` | แค่ atomicity ของการดำเนินการเดียว ไม่รับประกันลำดับกับตัวแปรอื่น | เร็วที่สุด | ตัวนับสถิติล้วนๆ (เช่น request counter, log counter) ที่ไม่ต้อง sync กับข้อมูลอื่น |
| `memory_order_acquire`/`release` | สร้าง happens-before ระหว่าง thread ผ่านตัวแปร atomic ตัวเดียว | เร็วกว่า seq_cst | รูปแบบ producer-consumer แบบง่าย, flag บอกสถานะพร้อม/ไม่พร้อม |
| `memory_order_seq_cst` (default) | ลำดับเดียวกันทั้งโปรแกรมสำหรับทุก thread | ช้าที่สุดใน 3 แบบนี้ (แต่ยังเร็วกว่า mutex มาก) | ค่าเริ่มต้นที่ปลอดภัยที่สุด — **ใช้ค่านี้เป็น default เสมอถ้าไม่มั่นใจ** แล้วค่อย optimize เป็น relaxed/acquire-release ทีหลังเมื่อวัดผลแล้วว่าคุ้มค่าจริง |

> **คำแนะนำสำหรับผู้เริ่มต้น**: ในโค้ด production ทั่วไป **ให้ใช้ค่า default
> (`seq_cst`) ไปก่อนเสมอ** จนกว่าจะพิสูจน์ได้ด้วยการ Profile (Part 86) ว่า memory
> ordering คือคอขวดจริงๆ ของโปรแกรม การไล่ optimize เป็น `relaxed`/`acquire-release`
> ก่อนเวลาอันควรเป็นสาเหตุของบั๊กที่ตรวจจับยากที่สุดประเภทหนึ่งในโปรแกรม concurrent —
> เนื้อหาในหัวข้อนี้มีไว้เพื่อให้ **เข้าใจว่ากำลังเกิดอะไรขึ้นเวลาอ่านโค้ดคนอื่น** มากกว่า
> ให้รีบนำไปใช้เองทันที

---

## 82.5 `std::lock()` และ `std::scoped_lock`: ล็อกหลาย Mutex อย่างปลอดภัยจาก Deadlock (Step 654–655)

### ทบทวนปัญหาจาก Part 32: Deadlock จาก Lock Ordering ที่ไม่ตรงกัน

Part 32 หัวข้อ 32.7 สาธิตให้เห็นว่าการล็อก 2 mutex **คนละลำดับกัน** ระหว่าง 2 thread
ทำให้เกิด Deadlock ได้ทันที และหัวข้อ 32.8 แก้ปัญหานี้ด้วยการ**กำหนด Lock Ordering ให้
ตายตัวและเหมือนกันทุก thread ด้วยมือ** ปัญหาคือวิธีนี้ต้อง**อาศัยวินัยของโปรแกรมเมอร์
ทุกคนในทีม** ให้ทำตามกฎเดียวกันเป๊ะๆ ไม่มีอะไรบังคับในระดับภาษา

มาดูปัญหาเดียวกันในสไตล์ `std::mutex` ก่อน:

```cpp
#include <chrono>
#include <iostream>
#include <mutex>
#include <thread>

struct Account {
    int balance;
    std::mutex m;
    explicit Account(int b) : balance(b) {}
};

// เวอร์ชันไม่ดี: ล็อกทีละตัวตรงๆ ตามลำดับ parameter -- เกิด deadlock ได้เหมือน Part 32
void transfer_naive(Account& from, Account& to, int amount) {
    std::lock_guard<std::mutex> lock1(from.m);
    std::this_thread::sleep_for(std::chrono::milliseconds(100)); // ขยายช่องโหว่ให้เห็นชัด
    std::lock_guard<std::mutex> lock2(to.m);
    from.balance -= amount;
    to.balance += amount;
    std::cout << "[transfer_naive] โอน " << amount << " สำเร็จ\n";
}

int main() {
    Account a(1000);
    Account b(1000);

    std::thread t1(transfer_naive, std::ref(a), std::ref(b), 100); // ล็อก a ก่อน b
    std::thread t2(transfer_naive, std::ref(b), std::ref(a), 200); // ล็อก b ก่อน a -- สลับกัน!

    t1.join();
    t2.join();

    std::cout << "[main] จบโปรแกรม (ไม่ควรมาถึงถ้า deadlock)\n";
    return 0;
}
```

ถ้าเรียก `transfer_naive(a, b, 100)` จาก thread หนึ่ง และ `transfer_naive(b, a, 200)`
จากอีก thread หนึ่งพร้อมกัน (ล็อคกันคนละลำดับ) โปรแกรมจะค้างทันที:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread naive_deadlock.cpp -o naive_deadlock
timeout 3 ./naive_deadlock; echo "EXIT: $?"
```

```
EXIT: 124
```

`124` คือ exit code ของคำสั่ง `timeout` เมื่อโปรแกรมไม่จบภายในเวลาที่กำหนด — ยืนยันว่า
เกิด Deadlock จริงตามที่คาดไว้ เหมือนกับ `deadlock.c` ใน Part 32 ทุกประการ

### ทางแก้ที่ Modern C++ เพิ่มเข้ามา: `std::lock()`

`std::lock()` (มีมาตั้งแต่ C++11) รับ mutex **หลายตัว** เป็น argument แล้วล็อกให้**ทั้ง
หมดพร้อมกันแบบ all-or-nothing** โดยใช้ algorithm ภายใน (เช่น deadlock avoidance ผ่าน
`try_lock` สลับกันไปมา) ที่**รับประกันไม่มี Deadlock** ไม่ว่าผู้เรียกจะส่ง mutex มาคน
ละลำดับกันแค่ไหนก็ตาม — เราไม่ต้องกำหนด Lock Ordering เองด้วยมือเหมือน Part 32 อีก
ต่อไป:

```cpp
void transfer(Account& from, Account& to, int amount) {
    // วิธีก่อน C++17: std::lock() ล็อกหลาย mutex พร้อมกันแบบไม่ deadlock
    // แล้วใช้ unique_lock + std::adopt_lock เพื่อ "รับช่วง" การถือ lock ที่ล็อกไปแล้ว
    std::lock(from.m, to.m);
    std::unique_lock<std::mutex> lock_from(from.m, std::adopt_lock);
    std::unique_lock<std::mutex> lock_to(to.m, std::adopt_lock);

    from.balance -= amount;
    to.balance += amount;
}
```

`std::adopt_lock` บอก `unique_lock` ว่า **"mutex ตัวนี้ถูกล็อกไปแล้วโดย `std::lock()`
ก่อนหน้านี้ ไม่ต้องล็อกซ้ำ แค่ช่วยจดจำไว้เพื่อปลดล็อกให้อัตโนมัติตอน destruct"**

### วิธีที่แนะนำใน C++17: `std::scoped_lock`

C++17 เพิ่ม **`std::scoped_lock`** ที่ทำหน้าที่เดียวกับ `std::lock()` +
`std::adopt_lock` ทั้งหมดในบรรทัดเดียว อ่านง่ายกว่ามาก และเป็นวิธีที่แนะนำที่สุดสำหรับ
โค้ดใหม่ที่ใช้ C++17 ขึ้นไป:

```cpp
#include <iostream>
#include <mutex>
#include <thread>

struct Account {
    int balance;
    std::mutex m;
    explicit Account(int b) : balance(b) {}
};

// โอนเงินระหว่างสองบัญชี ต้องล็อกทั้งคู่พร้อมกันแบบปลอดภัยจาก deadlock
void transfer(Account& from, Account& to, int amount) {
    // std::scoped_lock (C++17) ล็อกหลาย mutex พร้อมกันแบบ "all-or-nothing"
    // ใช้ algorithm ภายในป้องกัน deadlock แม้ผู้เรียกจะส่ง mutex มาคนละลำดับกันก็ตาม
    // (ต่างจาก Part 32 ที่ต้องกำหนด Lock Ordering เองด้วยมือ)
    std::scoped_lock lock(from.m, to.m);
    from.balance -= amount;
    to.balance += amount;
    std::cout << "[transfer] โอน " << amount << " สำเร็จ\n";
}

int main() {
    Account a(1000);
    Account b(1000);

    // จงใจสลับลำดับ argument ระหว่าง 2 thread เหมือนสถานการณ์ deadlock ใน Part 32
    // แต่เพราะใช้ scoped_lock แทน lock ทีละตัวตรงๆ จึงไม่มีทาง deadlock เกิดขึ้นได้เลย
    std::thread t1(transfer, std::ref(a), std::ref(b), 100);
    std::thread t2(transfer, std::ref(b), std::ref(a), 200);

    t1.join();
    t2.join();

    std::cout << "[main] a.balance = " << a.balance << ", b.balance = " << b.balance << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread scoped_lock_demo.cpp -o scoped_lock_demo
./scoped_lock_demo
```

ผลลัพธ์ (รันซ้ำกี่ครั้งก็ไม่ deadlock เลยสักครั้ง แม้จะสลับลำดับ argument จงใจ):

```
[transfer] โอน 100 สำเร็จ
[transfer] โอน 200 สำเร็จ
[main] a.balance = 1100, b.balance = 900
```

### เทียบ `std::scoped_lock` กับวิธี Lock Ordering ของ Part 32

| หัวข้อ | Lock Ordering ด้วยมือ (Part 32) | `std::scoped_lock` (Part นี้) |
|---|---|---|
| ต้องกำหนดกฎเรื่องลำดับการล็อกเองไหม | ต้อง (เช่น "ล็อก address ต่ำกว่าก่อนเสมอ") และทุกคนในทีมต้องรู้และทำตามกฎเดียวกัน | ไม่ต้อง — ส่ง mutex เข้าไปในลำดับใดก็ได้ ปลอดภัยเสมอ |
| ความเสี่ยงจากมนุษย์ลืมทำตามกฎ | มี (ถ้ามีคนในทีมเขียนโค้ดที่ล็อกผิดลำดับแม้แต่จุดเดียว ก็ deadlock ได้) | ไม่มี — ตัว library รับประกันให้ในทุกจุดที่เรียกใช้ |
| ใช้ได้กับ pthread ไหม | ได้ (`pthread_mutex_t` ธรรมดา) | ไม่ได้โดยตรง (เป็นของ `std::mutex` เท่านั้น) |
| จำนวนบรรทัดโค้ด | มากกว่า (ต้อง comment อธิบายกฎ, มักต้องมี code review ตรวจสอบ) | บรรทัดเดียว `std::scoped_lock lock(m1, m2, m3, ...);` รองรับกี่ mutex ก็ได้ |

`std::scoped_lock` คือตัวอย่างที่ชัดเจนอีกครั้งของปรัชญา Modern C++ ที่ Part 80 สรุปไว้:
**ย้ายภาระด้านความถูกต้องจากวินัยของมนุษย์ไปให้ Type System และ Standard Library
จัดการแทน** เหมือนกับที่ RAII ย้ายภาระเรื่อง unlock ไปให้ destructor และเหมือนกับที่
`std::vector` ย้ายภาระเรื่อง memory management ไปจาก raw pointer ใน Part 80 หัวข้อ 80.5

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ล็อก `std::mutex` ธรรมดาซ้ำสองครั้งในเธรดเดียวกัน** — เกิด Deadlock กับตัวเองทันที
   เหมือนกับ `PTHREAD_MUTEX_NORMAL` ใน Part 32 มักเกิดจากฟังก์ชัน recursive ที่ถือ
   lock อยู่แล้วเรียกตัวเองซ้ำ หรือฟังก์ชันสองตัวที่ต่างก็ lock mutex เดียวกันแล้วเรียก
   กันเอง วิธีแก้คือใช้ `std::recursive_mutex` (ถ้าจำเป็นจริงๆ) หรือปรับโครงสร้างโค้ด
   ให้ไม่ต้อง lock ซ้ำเลย

2. **ใช้ `std::atomic` ป้องกัน Critical Section ที่ครอบหลายตัวแปร/หลายคำสั่ง** —
   `std::atomic` ป้องกันได้แค่**การดำเนินการเดียว**บนตัวแปรเดียวเท่านั้น ถ้าต้องแก้ไข
   หลายตัวแปรให้สอดคล้องกัน (เช่น ย้ายเงินระหว่างบัญชี) ต้องใช้ `std::mutex` เท่านั้น
   ห้ามคิดว่าทำให้ทุกตัวแปรที่เกี่ยวข้องเป็น `atomic` แยกกันแล้วจะปลอดภัย — ระหว่างการ
   แก้ตัวแปรที่ 1 เสร็จกับตัวแปรที่ 2 ยังไม่เริ่ม อาจมี thread อื่นเข้ามาเห็นสถานะที่ยัง
   ไม่สมบูรณ์ได้เสมอ

3. **ล็อก mutex หลายตัวทีละตัวตรงๆ (ไม่ใช้ `std::lock`/`std::scoped_lock`) โดยลำดับ
   ไม่ตรงกันระหว่างจุดเรียกต่างๆ** — สาเหตุของ Deadlock ที่พบบ่อยที่สุด สาธิตให้เห็น
   ชัดเจนใน 82.5 (`naive_deadlock.cpp`) แก้ไขได้ทันทีด้วย `std::scoped_lock`

4. **ใช้ `memory_order_relaxed` ทั้งที่ต้อง synchronize ข้อมูลอื่นด้วย** — ถ้าใช้
   `relaxed` กับตัวแปรที่ทำหน้าที่เป็น "สัญญาณ" บอกว่าข้อมูลอื่นพร้อมแล้ว (เหมือน
   `data_ready` ใน 82.4) โดยไม่ใช้ acquire/release จะเกิด Race Condition ที่ตรวจจับได้
   ยากมาก เพราะอาจทำงานถูกต้องเกือบทุกครั้งที่รัน (ขึ้นกับ CPU architecture และระดับ
   optimization) แล้วพังแบบสุ่มเฉพาะบางสถานการณ์เท่านั้น

5. **คิดว่า `std::atomic` "เร็วกว่า mutex เสมอ" แล้วเลือกใช้โดยไม่พิจารณาบริบท** —
   Atomic เหมาะกับข้อมูลเดี่ยวๆ ง่ายๆ เท่านั้น การพยายามสร้าง data structure ที่ซับซ้อน
   ด้วย atomic ล้วนๆ (Lock-Free Programming เต็มรูปแบบ) เป็นเรื่องยากและเสี่ยงบั๊กสูงมาก
   กว่าที่คิด (จะเรียนเจาะลึกความซับซ้อนนี้ใน Part 84)

6. **ลืมว่า `mutable` จำเป็นสำหรับ mutex ที่ใช้ใน const member function** — ถ้าต้อง
   lock mutex ภายใน method ที่ประกาศเป็น `const` (เช่น getter อย่าง `.balance()` ใน
   แบบฝึกหัดข้อ 1) ต้องประกาศ `std::mutex` เป็น `mutable` เสมอ ไม่งั้นจะ compile error
   เพราะ `const` method ห้ามแก้ไข non-mutable member ใดๆ (รวมถึงการเรียก `.lock()` ที่
   แก้ไข internal state ของ mutex เอง)

---

## แบบฝึกหัดท้ายบท

1. เขียน class `BankAccount` ที่มี `deposit()`, `withdraw()`, และ `balance()` ป้องกัน
   Race Condition ด้วย `std::mutex` + `std::lock_guard` แล้วทดสอบด้วยหลาย thread ที่
   ฝากเงินพร้อมกัน

2. ทดลองรัน `naive_deadlock.cpp` จากหัวข้อ 82.5 ด้วยตัวเอง (ควรจะค้าง) แล้วแก้ไขด้วย
   `std::scoped_lock` ให้ทำงานถูกต้องโดยไม่ deadlock

3. เขียนโปรแกรมที่ล็อก `std::mutex` ธรรมดาซ้ำสองครั้งในฟังก์ชันเดียวกัน (จงใจสร้าง
   deadlock) แล้วรันด้วย `timeout 3` เพื่อยืนยันว่ามันค้างจริง จากนั้นแก้ไขด้วย
   `std::recursive_mutex`

4. แก้ไข `memory_order_relaxed.cpp` ให้ใช้ `memory_order_seq_cst` (default) แทน แล้ว
   วัดเวลาเปรียบเทียบทั้งสองเวอร์ชัน อธิบายว่าทำไมความแตกต่างอาจจะน้อยมากในเครื่องที่ใช้
   ทดสอบ (ใบ้: ลองคิดว่า x86 กับ ARM มีพฤติกรรมต่างกันแค่ไหนในเรื่องนี้)

5. เขียนโปรแกรมเปรียบเทียบเวลาเมื่อ `INCREMENTS_PER_THREAD` เพิ่มขึ้นเป็น 10 เท่า
   (10,000,000 ครั้งต่อ thread) ระหว่างเวอร์ชัน mutex กับ atomic แล้วรายงานอัตราส่วน
   ความเร็วที่ต่างออกไปจากตัวอย่างใน 82.3 หรือไม่ อย่างไร

6. อธิบายด้วยคำพูดตัวเอง (เขียนเป็น comment ในโค้ด) ว่าทำไมตัวอย่าง `acquire_release.cpp`
   ใน 82.4 ถึง **จำเป็นต้อง** ใช้ busy-wait loop (`while (!data_ready.load(...))`) ซึ่ง
   สิ้นเปลือง CPU แทนที่จะใช้ `std::condition_variable` เหมือนที่ Part 32 สอนไว้ — ลอง
   เขียนเวอร์ชันที่ใช้ `std::condition_variable` แทนดูว่าทำได้หรือไม่ (ใบ้: ต้องใช้คู่กับ
   `std::mutex`/`std::unique_lock` ไม่ใช่ `std::atomic` ล้วนๆ)

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>

class BankAccount {
public:
    explicit BankAccount(int initial_balance) : balance_(initial_balance) {}

    void deposit(int amount) {
        std::lock_guard<std::mutex> lock(mutex_);
        balance_ += amount;
    }

    bool withdraw(int amount) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (amount > balance_) {
            return false;
        }
        balance_ -= amount;
        return true;
    }

    int balance() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return balance_;
    }

private:
    int balance_;
    mutable std::mutex mutex_; // mutable เพราะต้อง lock ได้แม้ใน const member function
};

int main() {
    BankAccount account(0);
    constexpr int NUM_THREADS = 8;
    constexpr int DEPOSITS_PER_THREAD = 100000;

    std::vector<std::thread> threads;
    for (int i = 0; i < NUM_THREADS; ++i) {
        threads.emplace_back([&account]() {
            for (int j = 0; j < DEPOSITS_PER_THREAD; ++j) {
                account.deposit(1);
            }
        });
    }
    for (auto& t : threads) {
        t.join();
    }

    std::cout << "Expected: " << NUM_THREADS * DEPOSITS_PER_THREAD << "\n";
    std::cout << "Actual:   " << account.balance() << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread ex1_bank.cpp -o ex1_bank
./ex1_bank
```

ผลลัพธ์:

```
Expected: 800000
Actual:   800000
```

**อธิบาย**: `mutex_` ถูกประกาศเป็น `mutable` เพื่อให้ method `balance() const` เรียก
`.lock()`/`.unlock()` ผ่าน `lock_guard` ได้ (การ lock/unlock เป็นการแก้ไข internal
state ของ mutex เอง ซึ่งขัดกับ `const` โดยหลักการถ้าไม่มี `mutable` กำกับ) การออกแบบ
นี้ตรงกับหลักการ **Logical Constness** ที่กล่าวถึงเป็นนัยใน Part 43 — `balance()` เป็น
`const` ในความหมายว่า **"ไม่เปลี่ยนแปลงค่ายอดเงินที่มองเห็นจากภายนอก"** แม้ภายในจะต้อง
แก้ไข state ของ mutex ชั่วคราวระหว่างการอ่านก็ตาม

### แนวทางเฉลยข้อ 2

```cpp
#include <iostream>
#include <mutex>
#include <thread>

struct Account {
    int balance;
    std::mutex m;
    explicit Account(int b) : balance(b) {}
};

// แก้ naive_deadlock.cpp ด้วย std::scoped_lock (C++17)
void transfer_fixed(Account& from, Account& to, int amount) {
    std::scoped_lock lock(from.m, to.m); // ล็อกทั้งคู่แบบ all-or-nothing ไม่มีทาง deadlock
    from.balance -= amount;
    to.balance += amount;
    std::cout << "[transfer_fixed] โอน " << amount << " สำเร็จ\n";
}

int main() {
    Account a(1000);
    Account b(1000);

    // จงใจสลับลำดับเหมือนเดิม แต่คราวนี้ไม่ deadlock อีกต่อไป
    std::thread t1(transfer_fixed, std::ref(a), std::ref(b), 100);
    std::thread t2(transfer_fixed, std::ref(b), std::ref(a), 200);

    t1.join();
    t2.join();

    std::cout << "[main] a.balance = " << a.balance << ", b.balance = " << b.balance << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread ex2_fixed.cpp -o ex2_fixed
timeout 3 ./ex2_fixed; echo "EXIT: $?"
```

ผลลัพธ์ (รันสำเร็จทุกครั้ง ไม่มี deadlock เลย):

```
[transfer_fixed] โอน 100 สำเร็จ
[transfer_fixed] โอน 200 สำเร็จ
[main] a.balance = 1100, b.balance = 900
EXIT: 0
```

**อธิบาย**: เทียบกับ `naive_deadlock.cpp` ที่ล็อก `from.m` แล้วค่อยล็อก `to.m` แยกกัน
สองบรรทัด (เปิดช่องให้ thread อื่นแทรกเข้ามาล็อกสลับลำดับได้ระหว่างสองบรรทัดนั้นพอดี)
`std::scoped_lock lock(from.m, to.m);` ล็อกทั้งคู่ **ในการดำเนินการเดียวที่แบ่งแยกไม่ได้
ในเชิงตรรกะ** (atomic ในความหมายของการล็อก ไม่ใช่ `std::atomic<T>`) — ไม่มีช่วงเวลาใด
เลยที่ thread หนึ่งถือแค่ mutex ตัวเดียวแล้วรออีกตัว จึงไม่มีทาง Circular Wait (เงื่อนไข
ข้อที่ 4 ของ Coffman Conditions จาก Part 32) เกิดขึ้นได้เลย ไม่ว่าจะเรียกด้วยลำดับ
argument แบบใดก็ตาม

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- นำหลักการ RAII จาก Part 68 มาผูกกับ mutex ผ่าน `std::lock_guard` และ `std::unique_lock`
  แก้ปัญหา "ลืม unlock" ที่เคยเจอกับ pthread ใน Part 32 ได้อย่างสมบูรณ์
- รู้จัก `std::recursive_mutex` และ `std::timed_mutex` ซึ่งเทียบเท่ากับ
  `PTHREAD_MUTEX_RECURSIVE` และ `pthread_mutex_trylock` ตามลำดับ
- เข้าใจว่า `std::atomic<T>` คืออะไร ทำงานผ่าน CPU instruction ระดับ hardware โดยตรง
  และ**วัดความเร็วจริง**ที่เร็วกว่า mutex ประมาณ 3-4 เท่าสำหรับตัวนับธรรมดา
- ทำความเข้าใจ **C++ Memory Model** เบื้องต้นผ่าน `memory_order_relaxed`,
  `acquire`/`release`, และ `seq_cst` ในระดับที่นำไปอ่านโค้ดคนอื่นและตัดสินใจใช้งานได้จริง
- ใช้ `std::lock()` และ `std::scoped_lock` (C++17) ล็อกหลาย mutex พร้อมกันได้อย่าง
  ปลอดภัยจาก Deadlock โดยไม่ต้องพึ่งวินัยของมนุษย์ในการกำหนด Lock Ordering เองเหมือนที่
  Part 32 ต้องทำ

Part นี้ปิดท้ายเครื่องมือหลักสำหรับป้องกัน Race Condition แบบ **synchronous** (เขียน
ข้อมูลร่วมกันแบบต้องรอคิว) แต่ยังมีสถานการณ์ที่พบบ่อยมากอีกแบบหนึ่งที่เรายังไม่ได้แตะเลย:
**การรันงานแบบ asynchronous แล้วรอรับผลลัพธ์กลับมาทีหลัง** โดยไม่ต้องสร้าง thread,
shared variable, และ mutex ด้วยมือเองทั้งหมดตั้งแต่ต้น ใน **Part 83** เราจะเรียนรู้
**`std::future`, `std::async`, และ `std::promise`** ซึ่งเป็นเครื่องมือระดับสูงกว่าที่ห่อ
ความซับซ้อนของ thread+mutex ไว้ให้เกือบทั้งหมด รวมถึงวิธีที่ exception จาก
asynchronous task ถูกส่งข้าม thread กลับมาให้จัดการได้อย่างปลอดภัย (แก้ปัญหาที่ Part 81
หัวข้อ 81.5 ทิ้งไว้ว่า exception ข้าม thread ธรรมดาทำไม่ได้)

**ต่อไป:** [Part 83 — std::future, std::async, std::promise](./part-083-async-future-promise.md)
