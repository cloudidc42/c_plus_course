# Part 83: std::future, std::async, std::promise (Step 657–664)

> Module G — Concurrency และ Performance Engineering | Part 83 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 657–664
> Part ก่อนหน้า: [Part 82 — std::mutex, std::atomic, C++ Memory Model](./part-082-mutex-atomic-memory-model.md) | Part ถัดไป: [Part 84 — Lock-Free Programming เบื้องต้น](./part-084-lock-free.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าปัญหาอะไรที่ `std::future`/`std::async` แก้ ที่การใช้ `std::thread` +
   ตัวแปรร่วม + mutex ตรงๆ จาก Part 81-82 ยังทำได้ไม่สะดวก
2. ใช้ `std::async` รันงานแบบ asynchronous พร้อมเลือก Launch Policy
   (`std::launch::async` กับ `std::launch::deferred`) ได้อย่างถูกต้องและอธิบายความ
   แตกต่างของพฤติกรรมได้
3. ใช้ `std::future::get()` และ `.wait()`/`.wait_for()` ดึงผลลัพธ์จาก asynchronous
   task ได้อย่างถูกต้อง พร้อมรู้ข้อจำกัดสำคัญที่สุดของ `std::future`
4. ใช้ `std::promise` ส่งค่าข้ามจาก thread หนึ่งไปยังอีก thread หนึ่งแบบ manual เมื่อ
   ต้องการควบคุมมากกว่าที่ `std::async` ให้ได้
5. ใช้ `std::packaged_task` ห่อฟังก์ชันธรรมดาให้ผูกกับ `std::future` โดยอัตโนมัติ
6. จัดการ Exception ที่เกิดขึ้นใน asynchronous task ได้อย่างถูกต้อง เข้าใจว่า exception
   ถูกเก็บไว้ใน future อย่างไรและถูก rethrow ตอนไหน (แก้ข้อจำกัดที่ Part 81 หัวข้อ 81.5
   ทิ้งไว้ว่า exception ข้าม thread ธรรมดาทำไม่ได้)
7. เขียนโปรแกรมจริงที่แบ่งงานคำนวณหนักออกเป็นหลายส่วนทำงานแบบขนานด้วย `std::async`
   แล้วรวมผลลัพธ์กลับมา พร้อมวัดผลเปรียบเทียบกับการทำงานแบบเรียงลำดับ

---

## 83.1 ปัญหาที่ `std::future`/`std::async` แก้ (Step 657)

ย้อนกลับไปดูสิ่งที่เราต้องทำใน Part 81-82 ทุกครั้งที่ต้องการ **"รันงานแบบขนานแล้วรอรับ
ผลลัพธ์กลับมา"**:

1. สร้าง `std::thread` ขึ้นมา
2. เตรียมตัวแปร (หรือ `vector`) สำหรับเก็บผลลัพธ์ที่ thread จะเขียนกลับมา
3. ส่ง reference ของตัวแปรนั้นผ่าน `std::ref` ให้ thread เขียนผลลัพธ์เข้าไป
4. `.join()` รอให้ thread ทำงานจบ
5. อ่านค่าจากตัวแปรที่เตรียมไว้
6. ถ้ามีหลาย thread เขียนคนละที่ ต้องระวังไม่ให้เขียนทับกัน (แม้จะคนละ index ก็ต้อง
   ออกแบบให้ถูกต้อง)
7. ถ้าเกิด exception ภายใน thread ต้องดักจับเองให้ครบ**ภายใน**ฟังก์ชันนั้นเลย เพราะ
   ส่งกลับมาให้ `main()` ดักจับไม่ได้เลย (Part 81 หัวข้อ 81.5)

นี่คือ "boilerplate" (โค้ดซ้ำๆ ที่ต้องเขียนทุกครั้ง) จำนวนมากสำหรับความต้องการที่พบบ่อย
มากในโปรแกรมจริง: **"ไปทำงานนี้ให้หน่อย เดี๋ยวฉันจะมาถามผลทีหลัง"**

`std::future` และ `std::async` (เพิ่มเข้ามาพร้อมกับ `std::thread` ใน C++11) ห่อ
ขั้นตอนทั้งหมดข้างต้นไว้ให้เกือบสมบูรณ์ — ไม่ต้องสร้าง thread เอง ไม่ต้องเตรียมตัวแปรร่วม
เอง ไม่ต้อง mutex ใดๆ เลย และที่สำคัญที่สุดคือ **exception ที่เกิดใน asynchronous task
จะถูก "ส่งกลับ" ให้ผู้เรียกจัดการได้อย่างปลอดภัย** ผ่านกลไกที่มาตรฐานกำหนดไว้ให้อัตโนมัติ
— แก้ข้อจำกัดสำคัญที่ Part 81 หัวข้อ 81.5 ทิ้งไว้ได้อย่างสมบูรณ์

### ภาพรวมความสัมพันธ์ของเครื่องมือใน Part นี้

```
std::async(func, args...)  ──────>  std::future<T>
      │                                   │
      │ รันงานบน thread ใหม่              │ .get() บล็อกรอผลลัพธ์
      │ (หรือ deferred แล้วรันตอน get)     │ (หรือ .wait()/.wait_for())
      ▼                                   ▼
  ทำงานเสร็จ ──── เก็บผลลัพธ์/exception ────> ส่งกลับผ่าน future

std::promise<T>  <───── set_value()/set_exception() จาก thread หนึ่ง
      │
      │ .get_future()
      ▼
std::future<T>  <───── .get() จากอีก thread หนึ่ง

std::packaged_task<T(Args...)>  = ห่อฟังก์ชันธรรมดา + ผูก future ให้อัตโนมัติ
```

ทั้งสามตัว (`std::async`, `std::promise`, `std::packaged_task`) ล้วนจบลงที่
**`std::future`** เหมือนกัน — ต่างกันแค่ **"วิธีที่ได้ future มา"** เท่านั้น

---

## 83.2 `std::async`: รันงานแบบ Asynchronous ในบรรทัดเดียว (Step 658)

```cpp
#include <future>
#include <iostream>
#include <thread>

int compute_square(int x) {
    std::cout << "[compute_square] กำลังคำนวณ " << x << "^2 บน thread " << std::this_thread::get_id() << "\n";
    return x * x;
}

int main() {
    std::cout << "[main] thread " << std::this_thread::get_id() << " เริ่มงาน\n";

    // std::async รันฟังก์ชันแบบ asynchronous แล้วคืน std::future<int> กลับมาทันที
    // ไม่ต้องสร้าง thread เอง ไม่ต้องสร้างตัวแปร shared ไม่ต้อง mutex ใดๆ ทั้งสิ้น
    std::future<int> fut = std::async(std::launch::async, compute_square, 7);

    std::cout << "[main] ระหว่างรอผลลัพธ์ ทำงานอย่างอื่นต่อได้เลย...\n";

    int result = fut.get(); // บล็อกรอผลลัพธ์ (ถ้ายังไม่เสร็จ) แล้วคืนค่ากลับมา
    std::cout << "[main] ผลลัพธ์ที่ได้: " << result << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread async_basic.cpp -o async_basic
./async_basic
```

ผลลัพธ์:

```
[main] thread 139988222691136 เริ่มงาน
[main] ระหว่างรอผลลัพธ์ ทำงานอย่างอื่นต่อได้เลย...
[compute_square] กำลังคำนวณ 7^2 บน thread 139988215789248
[main] ผลลัพธ์ที่ได้: 49
```

เทียบกับการเขียนด้วย `std::thread` + `std::ref` แบบ Part 81 ที่ต้องมี:

```cpp
int result = 0;
std::thread t([&result]() { result = compute_square(7); });
t.join();
// ใช้ result ต่อ
```

`std::async` ทำสิ่งเดียวกันแต่**สั้นกว่า**และ**ปลอดภัยกว่า** — ไม่ต้องประกาศตัวแปร
`result` แยก ไม่ต้อง capture by reference (ซึ่งเสี่ยง dangling reference ถ้าเขียนผิด
ตามที่ Part 81 เตือนไว้) และที่สำคัญที่สุด: **ถ้า `compute_square` throw exception
`std::async` จะเก็บ exception นั้นไว้ใน future ให้อัตโนมัติ** (รายละเอียดเต็มใน 83.6)

### Launch Policy: `std::launch::async` vs `std::launch::deferred`

`std::async` รับ parameter แรกเป็น **launch policy** ที่กำหนดว่างานจะถูกรัน "เมื่อไหร่"
และ "บน thread ไหน":

```cpp
#include <future>
#include <iostream>
#include <thread>

void show(const char* label) {
    std::cout << "[" << label << "] รันบน thread " << std::this_thread::get_id() << "\n";
}

int main() {
    std::cout << "[main] thread หลักคือ " << std::this_thread::get_id() << "\n\n";

    std::cout << "-- std::launch::async: บังคับให้รันบน thread ใหม่ทันที --\n";
    auto fut_async = std::async(std::launch::async, show, "async");
    fut_async.wait();

    std::cout << "\n-- std::launch::deferred: ยังไม่รันจนกว่าจะเรียก get()/wait() --\n";
    auto fut_deferred = std::async(std::launch::deferred, show, "deferred");
    std::cout << "[main] สร้าง future แบบ deferred แล้ว แต่ยังไม่รันฟังก์ชันเลย\n";
    fut_deferred.wait(); // ฟังก์ชันจะรันตรงนี้ "บน thread ที่เรียก wait/get" ไม่ใช่ thread ใหม่

    std::cout << "\n-- ไม่ระบุ policy (default): compiler เลือกเองระหว่าง async กับ deferred --\n";
    auto fut_default = std::async(show, "default");
    fut_default.wait();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread launch_policy.cpp -o launch_policy
./launch_policy
```

ผลลัพธ์:

```
[main] thread หลักคือ 140248745649984

-- std::launch::async: บังคับให้รันบน thread ใหม่ทันที --
[async] รันบน thread 140248738690752

-- std::launch::deferred: ยังไม่รันจนกว่าจะเรียก get()/wait() --
[main] สร้าง future แบบ deferred แล้ว แต่ยังไม่รันฟังก์ชันเลย
[deferred] รันบน thread 140248745649984

-- ไม่ระบุ policy (default): compiler เลือกเองระหว่าง async กับ deferred --
[default] รันบน thread 140248738690752
```

สังเกตว่า thread id ของ `[deferred]` **ตรงกับ main thread เป๊ะ** (`140248745649984`)
ในขณะที่ `[async]` รันบน thread ใหม่ที่มี id ต่างออกไป — นี่คือหลักฐานที่ยืนยันความหมาย
ของแต่ละ policy:

| Launch Policy | พฤติกรรม |
|---|---|
| `std::launch::async` | **บังคับ**ให้สร้าง thread ใหม่และเริ่มรันทันที ไม่ต้องรอเรียก `get()`/`wait()` |
| `std::launch::deferred` | **ไม่สร้าง thread ใหม่เลย** — ฟังก์ชันจะถูกเรียกก็ต่อเมื่อมีการเรียก `.get()` หรือ `.wait()` เท่านั้น และรันบน**thread เดียวกับที่เรียก get/wait นั้นเอง** (เป็น "lazy evaluation" ไม่ใช่ concurrency จริง) |
| ไม่ระบุ (default, `std::launch::async \| std::launch::deferred`) | **ไม่รับประกัน** — compiler/runtime เลือกเองว่าจะใช้แบบไหน ขึ้นกับจำนวน thread ที่มีอยู่และภาระของระบบขณะนั้น |

> **ข้อควรระวังสำคัญที่สุด**: การ**ไม่ระบุ launch policy**เลย (เขียนแค่
> `std::async(func, args...)`) เป็นพฤติกรรมที่ **คาดเดาไม่ได้แน่นอน** — ถ้าต้องการให้
> งานรันแบบขนานจริงเสมอ ต้องระบุ `std::launch::async` อย่างชัดเจนทุกครั้ง ไม่งั้นบาง
> ครั้ง runtime อาจเลือกใช้ `deferred` ทำให้โปรแกรมกลายเป็น sequential โดยไม่ตั้งใจ
> (โดยเฉพาะเมื่อระบบมีภาระสูงมากๆ) **หลักปฏิบัติของ Part นี้: ระบุ `std::launch::async`
> เสมอเมื่อต้องการ concurrency จริง**

---

## 83.3 `std::future::get()` และ `.wait()`/`.wait_for()` (Step 659)

### `.get()`: ดึงผลลัพธ์ (เรียกได้เพียงครั้งเดียวเท่านั้น!)

`.get()` บล็อกรอจนกว่างานจะเสร็จ แล้วคืนค่าผลลัพธ์กลับมา — แต่มีข้อจำกัดที่สำคัญที่สุด
ของ `std::future` ที่ต้องจำให้ขึ้นใจ: **เรียก `.get()` ได้เพียงครั้งเดียวเท่านั้น**
ต่อ future หนึ่งตัว

```cpp
#include <future>
#include <iostream>

int main() {
    std::future<int> fut = std::async(std::launch::async, [] { return 42; });

    std::cout << "[main] get() ครั้งที่ 1: " << fut.get() << "\n";

    try {
        int second = fut.get(); // BUG: เรียกซ้ำ - future นี้ถูก "ดึงค่า" ไปแล้วครั้งเดียวเท่านั้น
        std::cout << "[main] get() ครั้งที่ 2: " << second << "\n";
    } catch (const std::future_error& e) {
        std::cout << "[main] เรียก get() ซ้ำ throw std::future_error: " << e.what() << "\n";
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread get_twice.cpp -o get_twice
./get_twice
```

ผลลัพธ์:

```
[main] get() ครั้งที่ 1: 42
[main] เรียก get() ซ้ำ throw std::future_error: std::future_error: No associated state
```

**เหตุผลเชิงลึก**: `.get()` **ย้าย (move)** ผลลัพธ์ออกจาก future เมื่อเรียกสำเร็จ
(สำหรับ type ที่ move ได้) และทำให้ future นั้น**หมดสถานะ** (`valid() == false`) ทันที
— ครั้งที่สองที่เรียก `.get()` จึงพบว่าไม่มี "associated state" เหลืออยู่ให้ดึงค่าอีก
แล้ว จึง throw `std::future_error` ออกมาแทนที่จะคืนค่าขยะหรือ Undefined Behavior

> **ถ้าต้องการอ่านผลลัพธ์หลายครั้ง**: ใช้ **`std::shared_future<T>`** แทน
> `std::future<T>` (แปลงได้ผ่าน `.share()`) ซึ่งอนุญาตให้เรียก `.get()` ได้หลายครั้ง
> และแชร์ให้หลาย thread อ่านพร้อมกันได้ด้วย (คล้ายกับความสัมพันธ์ระหว่าง
> `std::unique_ptr` กับ `std::shared_ptr` ใน Part 67)

### `std::shared_future`: เมื่อหลาย Thread ต้องอ่านผลลัพธ์เดียวกัน

`std::shared_future<T>` **copy ได้** (ต่างจาก `std::future<T>` ที่ move ได้อย่างเดียว)
และอนุญาตให้เรียก `.get()` ได้**หลายครั้งจากหลาย thread พร้อมกัน** โดยไม่ throw
`std::future_error` เหมือนที่เจอใน `std::future` ธรรมดา:

```cpp
#include <future>
#include <iostream>
#include <thread>
#include <vector>

int compute() {
    return 100;
}

int main() {
    std::future<int> fut = std::async(std::launch::async, compute);
    std::shared_future<int> shared_fut = fut.share(); // แปลงให้ copy และ get() ได้หลายครั้ง

    std::vector<std::thread> readers;
    for (int i = 0; i < 3; ++i) {
        // capture shared_fut by value ได้เลยเพราะมัน copy ได้ (ต่าง std::future)
        readers.emplace_back([shared_fut, i]() {
            std::cout << "[reader " << i << "] ค่าที่ได้: " << shared_fut.get() << "\n";
        });
    }

    for (auto& t : readers) {
        t.join();
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread shared_future_demo.cpp -o shared_future_demo
./shared_future_demo
```

ผลลัพธ์ (ลำดับอาจสลับกันได้ แต่ทุก reader ได้ค่าที่ถูกต้องเสมอ):

```
[reader 0] ค่าที่ได้: 100
[reader 1] ค่าที่ได้: 100
[reader 2] ค่าที่ได้: 100
```

หลังจากเรียก `.share()` แล้ว `fut` (ตัว `std::future` เดิม) จะหมดสถานะไปทันที
(เหมือนกับที่ `.get()` ทำ) — ownership ของผลลัพธ์ถูกย้ายไปอยู่ในสถานะที่
`shared_future` ทุกสำเนาสามารถเข้าถึงร่วมกันได้อย่างปลอดภัย (ภายในมีกลไก reference
counting คล้าย `std::shared_ptr` ที่เรียนใน Part 67 คอยจัดการอายุของสถานะนั้นให้)

### `.wait()` และ `.wait_for()`: รอโดยไม่ดึงผลลัพธ์

`.wait()` บล็อกรอจนกว่างานจะเสร็จ **โดยไม่ดึงค่าออกมา** (เรียกซ้ำได้หลายครั้ง ไม่ทำให้
future หมดสถานะ) ส่วน `.wait_for(duration)` กำหนดเวลารอสูงสุดได้ คล้ายกับ
`std::timed_mutex::try_lock_for` ที่เรียนใน Part 82:

```cpp
#include <chrono>
#include <future>
#include <iostream>
#include <thread>

int slow_computation() {
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    return 99;
}

int main() {
    std::future<int> fut = std::async(std::launch::async, slow_computation);

    // wait_for ให้กำหนดเวลาสูงสุดที่จะรอ โดยไม่บล็อกค้างตลอดไปเหมือน get()/wait()
    while (fut.wait_for(std::chrono::milliseconds(100)) != std::future_status::ready) {
        std::cout << "[main] ยังไม่เสร็จ ระหว่างนี้ไปทำงานอย่างอื่นก่อน...\n";
    }

    std::cout << "[main] พร้อมแล้ว! ผลลัพธ์ = " << fut.get() << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread wait_for_demo.cpp -o wait_for_demo
./wait_for_demo
```

ผลลัพธ์:

```
[main] ยังไม่เสร็จ ระหว่างนี้ไปทำงานอย่างอื่นก่อน...
[main] ยังไม่เสร็จ ระหว่างนี้ไปทำงานอย่างอื่นก่อน...
[main] ยังไม่เสร็จ ระหว่างนี้ไปทำงานอย่างอื่นก่อน...
[main] ยังไม่เสร็จ ระหว่างนี้ไปทำงานอย่างอื่นก่อน...
[main] พร้อมแล้ว! ผลลัพธ์ = 99
```

`.wait_for()` คืนค่าเป็น `std::future_status` หนึ่งในสามแบบ:

| ค่า | ความหมาย |
|---|---|
| `std::future_status::ready` | งานเสร็จแล้ว เรียก `.get()` ได้ทันทีโดยไม่บล็อก |
| `std::future_status::timeout` | ยังไม่เสร็จภายในเวลาที่กำหนด |
| `std::future_status::deferred` | งานนี้เป็น `std::launch::deferred` และยังไม่เคยถูกเรียก (จะรันก็ต่อเมื่อเรียก `.get()`/`.wait()` แบบไม่มี timeout เท่านั้น) |

---

## 83.4 `std::promise`: ส่งค่าข้าม Thread แบบ Manual (Step 660)

`std::async` สะดวกมากเมื่อ "งาน" สามารถเขียนเป็นฟังก์ชันที่มี return value ตรงๆ ได้ แต่
บางสถานการณ์ต้องการควบคุมที่ละเอียดกว่านั้น เช่น thread ที่ทำงานหลายอย่างต่อเนื่องแล้ว
ค่อย "ส่งสัญญาณ" ผลลัพธ์กลับมาตอนใดตอนหนึ่งระหว่างทาง (ไม่ใช่แค่ตอน return)
**`std::promise`** ให้การควบคุมนี้โดยตรง

`std::promise<T>` กับ `std::future<T>` เป็นคู่กัน: `std::promise` คือฝั่ง **"เขียน"**
ผลลัพธ์ (`.set_value()`) และ `std::future` ที่ได้จาก `.get_future()` คือฝั่ง **"อ่าน"**
ผลลัพธ์นั้น (`.get()`):

```cpp
#include <future>
#include <iostream>
#include <thread>

void worker(std::promise<int> result_promise) {
    std::cout << "[worker] กำลังคำนวณ...\n";
    int computed = 6 * 7;
    result_promise.set_value(computed); // ส่งค่าข้าม thread กลับไปให้ future ฝั่งตรงข้าม
}

int main() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future(); // ผูก future เข้ากับ promise ตัวนี้

    // ต้อง std::move promise เข้าไปใน thread เพราะ promise copy ไม่ได้ (ย้าย ownership เท่านั้น)
    std::thread t(worker, std::move(prom));

    std::cout << "[main] รอผลลัพธ์จาก promise ผ่าน future...\n";
    int value = fut.get(); // บล็อกรอจนกว่า worker จะเรียก set_value()
    std::cout << "[main] ได้ค่า: " << value << "\n";

    t.join();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread promise_basic.cpp -o promise_basic
./promise_basic
```

ผลลัพธ์:

```
[main] รอผลลัพธ์จาก promise ผ่าน future...
[worker] กำลังคำนวณ...
[main] ได้ค่า: 42
```

### `std::promise` vs `std::async`: เมื่อไหร่ควรใช้อะไร

| หัวข้อ | `std::async` | `std::promise` (คู่กับ `std::thread` เอง) |
|---|---|---|
| ความสะดวก | สูงมาก — บรรทัดเดียวได้ทั้ง thread และ future | ต้องสร้าง `std::thread` เอง + `std::promise` เอง แยกกัน |
| การควบคุม | จำกัด (ผลลัพธ์มาจาก return value ของฟังก์ชันเท่านั้น) | ยืดหยุ่นเต็มที่ — เรียก `.set_value()` จากจุดใดก็ได้ภายใน thread แม้จะอยู่ลึกในฟังก์ชันย่อยหลายชั้น |
| ใช้เมื่อ | งานที่เขียนเป็นฟังก์ชันธรรมดาที่มี return value ตรงไปตรงมา (ใช้บ่อยที่สุด) | ต้องการควบคุม thread เองอย่างละเอียด (เช่น ผูกกับ thread pool ที่มีอยู่แล้ว, หรือส่งค่าจากจุดกลางทางที่ซับซ้อน) |
| ต้อง `std::move` | ไม่เกี่ยว | ต้อง (`std::promise` copy ไม่ได้ เหมือน `std::thread`, `std::unique_ptr`) |

---

## 83.5 `std::packaged_task`: จุดกึ่งกลางระหว่าง `std::async` กับ `std::promise` (Step 661)

**`std::packaged_task`** ห่อฟังก์ชันธรรมดาให้ผูกกับ `std::future` โดยอัตโนมัติ —
ต่างจาก `std::promise` ตรงที่**ไม่ต้องเรียก `.set_value()` เอง** เรียก task เหมือน
ฟังก์ชันปกติ แล้วผลลัพธ์ (หรือ exception ถ้าเกิดขึ้น) จะถูกส่งเข้า future ให้อัตโนมัติ:

```cpp
#include <future>
#include <iostream>
#include <thread>

int add(int a, int b) {
    return a + b;
}

int main() {
    // std::packaged_task ห่อฟังก์ชันธรรมดาให้ผูกกับ future โดยอัตโนมัติ
    // ต่างจาก std::promise ตรงที่ไม่ต้องเรียก set_value() เอง - เรียก task เหมือนฟังก์ชันปกติ
    // แล้วผลลัพธ์ (หรือ exception) จะถูกส่งเข้า future ให้อัตโนมัติ
    std::packaged_task<int(int, int)> task(add);
    std::future<int> fut = task.get_future();

    // สามารถเรียกบน thread ใหม่ หรือเรียกตรงๆ บน thread ปัจจุบันก็ได้ ขึ้นกับการออกแบบ
    std::thread t(std::move(task), 3, 4);

    std::cout << "[main] ผลลัพธ์จาก packaged_task: " << fut.get() << "\n";

    t.join();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread packaged_task_demo.cpp -o packaged_task_demo
./packaged_task_demo
```

ผลลัพธ์:

```
[main] ผลลัพธ์จาก packaged_task: 7
```

### ตารางสรุปทั้งสามเครื่องมือ

| เครื่องมือ | ได้ future มาจาก | ใครเรียกฟังก์ชันจริง | เหมาะกับ |
|---|---|---|---|
| `std::async` | เรียกฟังก์ชันแล้วคืน future ทันที | Library จัดการให้ทั้งหมด (สร้าง thread เอง หรือ deferred) | งานทั่วไปที่เขียนเป็นฟังก์ชันธรรมดา — **ใช้บ่อยที่สุด** |
| `std::packaged_task` | `.get_future()` จาก task object | ผู้เขียนโค้ดเรียก task เอง (ผ่าน `operator()`) บน thread ใดก็ได้ที่ต้องการ | ต้องการควบคุมว่า "จะรันตอนไหน บน thread ไหน" เอง เช่น ผูกกับ thread pool ที่มีอยู่แล้ว (จะเจอแนวคิดนี้อีกครั้งใน Part 101) |
| `std::promise` | `.get_future()` จาก promise object | ผู้เขียนโค้ดเรียก `.set_value()`/`.set_exception()` เองจากจุดใดก็ได้ | ต้องการส่งผลลัพธ์จากจุดกลางทางที่ซับซ้อน ไม่ใช่แค่ return value ของฟังก์ชันเดียว |

---

## 83.6 Exception Handling ใน Async Task (Step 662)

นี่คือจุดที่ `std::future` แก้ปัญหาสำคัญที่ Part 81 หัวข้อ 81.5 ทิ้งไว้ได้อย่างสมบูรณ์:
**exception ที่เกิดขึ้นภายใน asynchronous task จะถูก "เก็บ" ไว้ในสถานะภายในของ future
โดยอัตโนมัติ แล้ว rethrow ออกมาตอนเรียก `.get()` พอดี** — เหมือนกับว่า exception นั้น
throw ออกมาจาก `.get()` เอง ทั้งที่จริงๆ แล้วมันเกิดขึ้นบน thread อื่นโดยสิ้นเชิง:

```cpp
#include <future>
#include <iostream>
#include <stdexcept>

int divide(int a, int b) {
    if (b == 0) {
        throw std::invalid_argument("หารด้วยศูนย์ไม่ได้");
    }
    return a / b;
}

int main() {
    std::future<int> fut = std::async(std::launch::async, divide, 10, 0);

    std::cout << "[main] เรียก get() เพื่อรอผลลัพธ์...\n";
    try {
        int result = fut.get(); // exception ที่เกิดใน async task ถูก "เก็บไว้" ใน future
                                 // แล้ว rethrow ออกมาตรงนี้พอดี เหมือนกับว่า throw ในตัว get() เอง
        std::cout << "[main] ผลลัพธ์: " << result << "\n";
    } catch (const std::invalid_argument& e) {
        std::cout << "[main] ดักจับ exception จาก async task ได้: " << e.what() << "\n";
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread async_exception.cpp -o async_exception
./async_exception
```

ผลลัพธ์:

```
[main] เรียก get() เพื่อรอผลลัพธ์...
[main] ดักจับ exception จาก async task ได้: หารด้วยศูนย์ไม่ได้
```

เทียบกับ Part 81 หัวข้อ 81.5 ที่ต้องดักจับ exception **ภายใน** thread function เอง
ให้ครบทุกกรณี (ไม่งั้น `std::terminate()` ทันที) ตัวอย่างนี้**ไม่มี** `try`/`catch`
ใดๆ อยู่ใน `divide()` เลย — exception ถูก throw ออกมาตามปกติ แล้ว **runtime ของ
`std::async` เป็นผู้ดักจับมันไว้แทน** เก็บไว้ในสถานะภายในของ future object จนกว่าจะมี
คนเรียก `.get()` แล้วจึง rethrow ออกมาให้ผู้เรียก `.get()` จัดการอีกที (บน thread ของ
ผู้เรียก ไม่ใช่ thread ที่ throw จริงๆ)

นี่คือเหตุผลสำคัญข้อหนึ่งที่ `std::async` ถูกแนะนำมากกว่าการใช้ `std::thread` ตรงๆ เมื่อ
เขียนโปรแกรมที่ต้องรับผลลัพธ์กลับมา — **การจัดการ exception ง่ายกว่ามาก และไม่มีความ
เสี่ยงที่จะลืมดักจับจนโปรแกรมพังทั้งกระบวนการ**

### กฎเดียวกันนี้ใช้ได้กับ `std::promise` และ `std::packaged_task` ด้วย

`std::promise` มี method คู่กับ `.set_value()` ชื่อ **`.set_exception()`** สำหรับส่ง
exception ข้าม thread แบบ manual:

```cpp
void worker(std::promise<int> prom) {
    try {
        throw std::runtime_error("เกิดปัญหาระหว่างทำงาน");
    } catch (...) {
        prom.set_exception(std::current_exception()); // เก็บ exception ปัจจุบันส่งผ่าน future
    }
}
```

`std::current_exception()` คืนค่า `std::exception_ptr` ที่ชี้ไปยัง exception ที่กำลัง
ถูกจัดการอยู่ (ต้องเรียกภายใน `catch` block เท่านั้น) ฝั่งที่เรียก `fut.get()` จะได้รับ
exception ตัวเดิมนี้ rethrow กลับมาให้จัดการทันที ส่วน `std::packaged_task` จัดการเรื่อง
นี้ให้อัตโนมัติเหมือน `std::async` ทุกประการ (ไม่ต้องเรียก `.set_exception()` เอง)

---

## 83.7 ตัวอย่างจริง: คำนวณงานหนักหลายส่วนแบบขนานแล้วรวมผลลัพธ์ (Step 663)

มาถึงตัวอย่างที่ครบวงจรที่สุดของ Part นี้: แบ่งงาน**นับจำนวนเฉพาะ (prime)** ในช่วง
ตัวเลขขนาดใหญ่ออกเป็นหลายส่วน ให้แต่ละส่วนทำงานแบบขนานผ่าน `std::async` แล้วรวมผลลัพธ์
กลับมาเปรียบเทียบกับการทำงานแบบเรียงลำดับ (sequential) — งานนับจำนวนเฉพาะด้วยวิธี
trial division เป็นตัวอย่างคลาสสิกของงาน CPU-bound ที่ **คุ้มค่าอย่างมาก** ที่จะแบ่งไป
ทำแบบขนาน:

```cpp
#include <chrono>
#include <cmath>
#include <future>
#include <iostream>
#include <vector>

// จำลองงานหนัก (CPU-bound): นับจำนวนเฉพาะ (prime) ในช่วง [start, end) ด้วยวิธี trial division
// ธรรมดา (ไม่ optimize) เพื่อให้เห็นภาพงานที่ "หนักจริง" คุ้มค่าที่จะแบ่งไปทำแบบขนาน
long long count_primes(int start, int end) {
    long long count = 0;
    for (int n = start; n < end; ++n) {
        if (n < 2) {
            continue;
        }
        bool is_prime = true;
        for (int d = 2; static_cast<long long>(d) * d <= n; ++d) {
            if (n % d == 0) {
                is_prime = false;
                break;
            }
        }
        if (is_prime) {
            ++count;
        }
    }
    return count;
}

int main() {
    constexpr int RANGE_END = 2'000'000;
    constexpr int NUM_TASKS = 4;
    const int chunk = RANGE_END / NUM_TASKS;

    // ---------- เวอร์ชันขนาน: แบ่งงานเป็น NUM_TASKS ส่วนด้วย std::async ----------
    auto parallel_start = std::chrono::steady_clock::now();

    std::vector<std::future<long long>> futures;
    for (int i = 0; i < NUM_TASKS; ++i) {
        int start = i * chunk;
        int end = (i == NUM_TASKS - 1) ? RANGE_END : (i + 1) * chunk;
        futures.push_back(std::async(std::launch::async, count_primes, start, end));
    }

    long long parallel_total = 0;
    for (auto& fut : futures) {
        parallel_total += fut.get();
    }

    auto parallel_end = std::chrono::steady_clock::now();
    auto parallel_ms =
        std::chrono::duration_cast<std::chrono::milliseconds>(parallel_end - parallel_start).count();

    // ---------- เวอร์ชันเรียงลำดับ: ทำทั้งหมดใน thread เดียวเพื่อเปรียบเทียบ ----------
    auto seq_start = std::chrono::steady_clock::now();
    long long sequential_total = count_primes(0, RANGE_END);
    auto seq_end = std::chrono::steady_clock::now();
    auto seq_ms = std::chrono::duration_cast<std::chrono::milliseconds>(seq_end - seq_start).count();

    std::cout << "จำนวนเฉพาะที่พบ (ขนาน):    " << parallel_total << "\n";
    std::cout << "จำนวนเฉพาะที่พบ (เรียงลำดับ): " << sequential_total << "\n";
    std::cout << "เวลาแบบขนาน (" << NUM_TASKS << " งาน): " << parallel_ms << " ms\n";
    std::cout << "เวลาแบบเรียงลำดับ:          " << seq_ms << " ms\n";
    std::cout << "เร็วขึ้นประมาณ: " << static_cast<double>(seq_ms) / static_cast<double>(parallel_ms)
              << " เท่า\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread -O2 parallel_sum.cpp -o parallel_sum
./parallel_sum
```

ผลลัพธ์ตัวอย่างจริงที่วัดได้ (ตัวเลขขึ้นกับสเปกเครื่องที่รัน):

```
จำนวนเฉพาะที่พบ (ขนาน):    148933
จำนวนเฉพาะที่พบ (เรียงลำดับ): 148933
เวลาแบบขนาน (4 งาน): 172 ms
เวลาแบบเรียงลำดับ:          384 ms
เร็วขึ้นประมาณ: 2.23 เท่า
```

### ทำไมเร็วขึ้นแค่ ~2.2 เท่า ทั้งที่ใช้ 4 งาน (ไม่ใช่ 4 เท่า)

สังเกตว่าแม้จะแบ่งงานเป็น 4 ส่วนเท่าๆ กัน (ตามจำนวนตัวเลขในแต่ละช่วง) แต่ความเร็วที่
ได้ **ไม่ใช่ 4 เท่า** เป๊ะ — นี่คือบทเรียนสำคัญเรื่อง **Load Balancing** ที่จะกลับมา
เจออีกครั้งใน Part 85 (Parallel Algorithm) และ Part 90 (Benchmarking): ต้นทุนของ
trial division ต่อตัวเลข `n` หนึ่งตัวคือประมาณ `O(sqrt(n))` — ตัวเลขในช่วงท้าย (เช่น
1.5-2 ล้าน) ใช้เวลาตรวจสอบต่อตัวมากกว่าตัวเลขในช่วงต้น (0-500,000) แม้จะมีจำนวนตัวเลข
เท่ากันในแต่ละ chunk ก็ตาม ทำให้ **งานของ task สุดท้ายหนักกว่า task แรกอย่างมีนัยสำคัญ**
— task ที่เบากว่าจะรอ task ที่หนักที่สุดให้เสร็จก่อนเสมอ (เพราะเราต้อง `.get()` ทุกตัว
ก่อนจะรวมผลได้) ทำให้ความเร็วโดยรวมไม่ได้ scale แบบเป็นเส้นตรงตามจำนวน task

นี่คือข้อจำกัดของการแบ่งงานแบบ **Static Partitioning** (แบ่งขนาดเท่ากันตายตัวล่วงหน้า)
ซึ่งง่ายต่อการเขียนแต่ไม่ได้ optimal เสมอไป — ทางแก้ที่ดีกว่าคือ **Dynamic
Partitioning** หรือใช้ Thread Pool ที่แจกงานเป็นชิ้นเล็กๆ ให้ thread ที่ว่างก่อน (แนวคิด
นี้จะเจอเต็มรูปแบบใน Part 101 ตอนสร้าง HTTP Server พร้อม Thread Pool) แต่สำหรับ Part
นี้ สิ่งสำคัญที่สุดที่ต้องจำคือ: **`std::async` ทำให้การกระจายงานเขียนง่ายมาก แต่การ
กระจาย "อย่างสมดุล" ยังคงต้องคิดและออกแบบเอง**

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เรียก `.get()` ซ้ำสองครั้งบน `std::future` เดียวกัน** — throw `std::future_error`
   ทันที เพราะ `.get()` ย้ายผลลัพธ์ออกไปและทำให้ future หมดสถานะหลังเรียกครั้งแรก ถ้า
   ต้องการอ่านผลลัพธ์หลายครั้งต้องแปลงเป็น `std::shared_future` ด้วย `.share()`

2. **ไม่เก็บ future ที่ได้จาก `std::async` ไว้เลย** — เป็นบั๊กที่พบบ่อยและอันตรายมาก
   เพราะ future ชั่วคราว (temporary) ที่ไม่ได้ถูกเก็บไว้ในตัวแปรจะถูกทำลายทันทีที่จบ
   statement และ **destructor ของ `std::future` ที่ได้จาก `std::async` (ไม่ใช่จาก
   `std::promise`) จะ "บล็อกรอ" ให้งานเสร็จก่อนเสมอ** (เหมือนเรียก `.wait()` ทันที)
   ทำให้ `std::async` กลายเป็น **synchronous** โดยไม่ตั้งใจ:

   ```cpp
   // ตั้งใจให้รันขนาน แต่บรรทัดนี้ "บล็อก" จนกว่า slow_task() จะเสร็จ เพราะ future
   // ชั่วคราวที่ไม่ได้เก็บไว้ถูก destruct ทันที และ destructor เรียก wait() ให้เอง
   std::async(std::launch::async, slow_task);
   std::cout << "บรรทัดนี้รันหลัง slow_task() เสร็จแล้วเท่านั้น (ทั้งที่ไม่ได้ตั้งใจ)\n";
   ```

   คอมไพเลอร์มักเตือนเรื่องนี้ด้วย `-Wunused-result` เพราะ `std::async` ถูกทำเครื่องหมาย
   `[[nodiscard]]` ไว้ในมาตรฐานเพื่อป้องกันบั๊กนี้โดยเฉพาะ **ต้องเก็บผลลัพธ์ของ
   `std::async` ไว้ในตัวแปรเสมอ** แม้จะไม่สนใจ return value ก็ตาม (อาจเก็บไว้ใน
   `std::future<void>` หรือ vector ของ future เพื่อ join ทีหลัง)

3. **ไม่ระบุ Launch Policy แล้วคาดหวังว่าจะรันแบบขนานเสมอ** — ถ้าไม่ใส่
   `std::launch::async` ชัดเจน runtime อาจเลือกใช้ `deferred` แทน ทำให้งานรันแบบ
   sequential บน thread ที่เรียก `.get()`/`.wait()` โดยไม่มีการแจ้งเตือนใดๆ ควรระบุ
   `std::launch::async` เสมอเมื่อต้องการ concurrency จริง

4. **สับสนระหว่าง `std::future` กับ `std::shared_future`** — `std::future` ย้าย
   ownership ได้อย่างเดียว (copy ไม่ได้ เหมือน `std::unique_ptr`) และเรียก `.get()`
   ได้ครั้งเดียว ในขณะที่ `std::shared_future` copy ได้และแชร์ให้หลาย thread เรียก
   `.get()` พร้อมกันได้หลายครั้ง — เลือกผิดจะทำให้เจอ compile error (พยายาม copy
   `std::future`) หรือ runtime error (`.get()` ซ้ำ) ตามแต่กรณี

5. **ลืมว่า `std::promise`/`std::packaged_task` copy ไม่ได้ ต้อง `std::move` เสมอเมื่อ
   ส่งเข้า `std::thread`** — เหมือนกับ `std::thread` และ `std::unique_ptr` ที่เรียน
   มาก่อนหน้านี้ ลืม `std::move` จะเจอ compile error ทันที (Part 70 อธิบาย Move
   Semantics ไว้อย่างละเอียดแล้ว หลักการเดียวกันนี้ใช้ได้กับทุก type ที่ "ห้าม copy"
   ในไลบรารี concurrency ทั้งหมด)

6. **สร้าง `std::async` จำนวนมากเกินไปโดยไม่คำนึงถึง `hardware_concurrency()`** —
   เหมือนที่ Part 81 เตือนไว้กับ `std::thread` ตรงๆ การสร้าง async task จำนวนมากกว่า
   จำนวน core จริงไม่ได้ทำให้เร็วขึ้นเสมอไป (แม้ `std::launch::async` อาจไม่ได้สร้าง
   thread ใหม่จริงเสมอไปในบาง implementation ที่ฉลาดพอจะมี thread pool ภายใน ก็ไม่ควร
   นับพึ่งพฤติกรรมนี้เพราะไม่ได้การันตีโดยมาตรฐาน)

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมที่ใช้ `std::async` คำนวณ Fibonacci ของตัวเลข 5 ค่าที่แตกต่างกันแบบขนาน
   (ใช้ `std::vector<std::future<long long>>` เก็บผลลัพธ์แต่ละตัว) แล้วพิมพ์ผลลัพธ์
   ทั้งหมดออกมาหลังจากรอครบทุกตัว

2. ทดลองรันโค้ดต่อไปนี้ (จงใจไม่เก็บ future) แล้ววัดเวลาด้วย `time` เปรียบเทียบกับ
   เวอร์ชันที่เก็บ future ไว้ในตัวแปร อธิบายผลลัพธ์ที่ได้:

   ```cpp
   std::async(std::launch::async, slow_task); // ไม่เก็บ future
   std::cout << "บรรทัดถัดไป...\n";
   ```

3. เขียนโปรแกรมที่ใช้ `std::promise`/`std::future` ส่ง exception ข้าม thread ด้วย
   `.set_exception(std::current_exception())` แล้วดักจับที่ฝั่ง `.get()`

4. แปลง `packaged_task_demo.cpp` ให้ใช้ฟังก์ชันที่ throw exception แทน (เช่น หารด้วย
   ศูนย์) แล้วพิสูจน์ว่า `std::packaged_task` ส่ง exception ผ่าน future ได้อัตโนมัติ
   เหมือนกับ `std::async` โดยไม่ต้องเรียก `.set_exception()` เอง

5. ปรับ `parallel_sum.cpp` (81.7) ให้แบ่งช่วงตัวเลขเป็น chunk เล็กลง (เช่น 16 chunks
   แทน 4) แล้ววัดว่าเวลาแบบขนานดีขึ้นหรือแย่ลง อธิบายผลลัพธ์โดยอ้างอิงเรื่อง Load
   Balancing ที่อธิบายไว้ใน 83.7

6. เขียนโปรแกรมที่แปลง `std::future<int>` เป็น `std::shared_future<int>` ด้วย
   `.share()` แล้วให้ 3 thread ต่างเรียก `.get()` พร้อมกันบน shared_future ตัวเดียวกัน
   พิสูจน์ว่าทำงานได้ถูกต้องโดยไม่มี exception เหมือนที่เกิดกับ `std::future` ธรรมดา

### แนวทางเฉลยข้อ 1

```cpp
#include <future>
#include <iostream>
#include <vector>

long long fibonacci(int n) {
    if (n <= 1) {
        return n;
    }
    long long a = 0;
    long long b = 1;
    for (int i = 2; i <= n; ++i) {
        long long next = a + b;
        a = b;
        b = next;
    }
    return b;
}

int main() {
    std::vector<int> inputs = {10, 20, 30, 40, 50};
    std::vector<std::future<long long>> futures;

    for (int n : inputs) {
        futures.push_back(std::async(std::launch::async, fibonacci, n));
    }

    for (size_t i = 0; i < inputs.size(); ++i) {
        std::cout << "fibonacci(" << inputs[i] << ") = " << futures[i].get() << "\n";
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread ex1_fib_async.cpp -o ex1_fib_async
./ex1_fib_async
```

ผลลัพธ์:

```
fibonacci(10) = 55
fibonacci(20) = 6765
fibonacci(30) = 832040
fibonacci(40) = 102334155
fibonacci(50) = 12586269025
```

**อธิบาย**: แต่ละ `std::async` ยิงงานออกไปทำงานแบบขนานทันที (`std::launch::async`)
เก็บ future ไว้ใน `vector` ตามลำดับเดียวกับ `inputs` จากนั้นวนลูป `.get()` ทีละตัว
ตามลำดับ index — แม้ thread แต่ละตัวจะทำงานเสร็จไม่พร้อมกัน แต่การพิมพ์ผลลัพธ์ยังคง
เรียงตามลำดับที่ต้องการเสมอ เพราะ `.get()` แต่ละตัวรอเฉพาะ future ของตัวเอง (ถ้า
future ตัวที่ 3 ยังไม่เสร็จตอนที่ลูปมาถึง index 3 โปรแกรมจะบล็อกรอตรงนั้นแค่ future
ตัวนั้นตัวเดียว)

### แนวทางเฉลยข้อ 3

```cpp
#include <future>
#include <iostream>
#include <stdexcept>
#include <thread>

void worker(std::promise<int> prom) {
    try {
        throw std::runtime_error("เกิดปัญหาระหว่างทำงานใน worker");
    } catch (...) {
        // เก็บ exception ปัจจุบันไว้ส่งผ่าน future แทนการปล่อยให้หลุดออกไป
        prom.set_exception(std::current_exception());
    }
}

int main() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future();

    std::thread t(worker, std::move(prom));

    try {
        int value = fut.get(); // exception จาก worker ถูก rethrow ตรงนี้
        std::cout << "[main] ได้ค่า: " << value << "\n";
    } catch (const std::runtime_error& e) {
        std::cout << "[main] ดักจับ exception จาก promise ได้: " << e.what() << "\n";
    }

    t.join();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread ex3_promise_exception.cpp -o ex3_promise_exception
./ex3_promise_exception
```

ผลลัพธ์:

```
[main] ดักจับ exception จาก promise ได้: เกิดปัญหาระหว่างทำงานใน worker
```

**อธิบาย**: `std::current_exception()` ต้องเรียก**ภายใน `catch` block เท่านั้น** —
มันคืนค่า `std::exception_ptr` ที่ "ห่อ" exception object ปัจจุบันไว้ในรูปแบบที่ส่ง
ข้าม thread ได้อย่างปลอดภัย (ต่างจาก exception object ดิบๆ ที่ผูกอยู่กับ call stack
ของ thread ที่ throw มันขึ้นมา ส่งข้าม thread ตรงๆ ไม่ได้) เมื่อ `prom.set_exception(...)`
ถูกเรียก ฝั่ง `fut.get()` จะ**เห็น exception ตัวเดิม** rethrow กลับมาให้จัดการ
เปรียบเสมือนว่า exception เดินทางข้าม thread ได้จริง แม้กลไก `try`/`catch` ปกติของ
C++ (ที่ Part 81 หัวข้อ 81.5 พิสูจน์ไปแล้วว่าทำแบบนั้นตรงๆ ไม่ได้) จะไม่รองรับก็ตาม —
นี่คือคุณค่าหลักของ `std::future`/`std::promise` ในเรื่อง exception safety

### แนวทางเฉลยข้อ 4

```cpp
#include <future>
#include <iostream>
#include <stdexcept>
#include <thread>

int divide(int a, int b) {
    if (b == 0) {
        throw std::invalid_argument("หารด้วยศูนย์ไม่ได้ (จาก packaged_task)");
    }
    return a / b;
}

int main() {
    std::packaged_task<int(int, int)> task(divide);
    std::future<int> fut = task.get_future();

    std::thread t(std::move(task), 10, 0);

    try {
        int result = fut.get();
        std::cout << "[main] ผลลัพธ์: " << result << "\n";
    } catch (const std::invalid_argument& e) {
        std::cout << "[main] packaged_task ส่ง exception ผ่าน future ได้อัตโนมัติ: "
                  << e.what() << "\n";
    }

    t.join();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread ex4_packaged_exception.cpp -o ex4_packaged_exception
./ex4_packaged_exception
```

ผลลัพธ์:

```
[main] packaged_task ส่ง exception ผ่าน future ได้อัตโนมัติ: หารด้วยศูนย์ไม่ได้ (จาก packaged_task)
```

**อธิบาย**: ต่างจากแบบฝึกหัดข้อ 3 ที่ต้องเรียก `prom.set_exception(std::current_exception())`
เองอย่างชัดเจนภายใน `catch` block โค้ดนี้**ไม่มี** `try`/`catch` อยู่ใน `divide()` เลย
แม้แต่บรรทัดเดียว — `std::packaged_task` ห่อการเรียกฟังก์ชันไว้ภายใน `operator()` ของ
มันเอง ถ้าฟังก์ชันที่ห่อไว้ throw exception ตัว `packaged_task` จะดักจับและเรียก
`.set_exception()` ให้อัตโนมัติเบื้องหลัง พฤติกรรมนี้เหมือนกับ `std::async` ทุกประการ
(ตามที่ 83.6 อธิบายไว้) เพราะทั้งคู่มีกลไกการจัดการนี้ในตัวอยู่แล้ว ต่างจาก
`std::promise` ที่เป็นเครื่องมือ "ระดับล่างกว่า" ที่ต้องทำทุกอย่างเองด้วยมือ

### แนวทางเฉลยข้อ 6

```cpp
#include <future>
#include <iostream>
#include <thread>
#include <vector>

int compute() {
    return 100;
}

int main() {
    std::future<int> fut = std::async(std::launch::async, compute);
    std::shared_future<int> shared_fut = fut.share(); // แปลงให้ copy และ get() ได้หลายครั้ง

    std::vector<std::thread> readers;
    for (int i = 0; i < 3; ++i) {
        readers.emplace_back([shared_fut, i]() {
            std::cout << "[reader " << i << "] ค่าที่ได้: " << shared_fut.get() << "\n";
        });
    }

    for (auto& t : readers) {
        t.join();
    }

    // เรียก get() อีกครั้งจาก main เองก็ยังทำได้ ไม่ throw เหมือน std::future ธรรมดา
    std::cout << "[main] เรียก get() ซ้ำจาก main ก็ยังได้ค่าเดิม: " << shared_fut.get() << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -pthread ex6_shared_future.cpp -o ex6_shared_future
./ex6_shared_future
```

ผลลัพธ์:

```
[reader 0] ค่าที่ได้: 100
[reader 1] ค่าที่ได้: 100
[reader 2] ค่าที่ได้: 100
[main] เรียก get() ซ้ำจาก main ก็ยังได้ค่าเดิม: 100
```

**อธิบาย**: การเรียก `shared_fut.get()` ทั้งหมด 4 ครั้ง (3 ครั้งจาก reader thread และ
อีก 1 ครั้งจาก `main`) **ไม่มีครั้งไหนเลย** ที่ throw `std::future_error` ต่างจาก
`std::future` ธรรมดาที่จะพังตั้งแต่การเรียกครั้งที่สอง — นี่คือความแตกต่างหลักที่ทำให้
`std::shared_future` เหมาะกับสถานการณ์ที่ผลลัพธ์ **หนึ่งค่าต้องถูกใช้ร่วมกันโดยหลาย
ส่วนของโปรแกรม** เช่น ค่า configuration ที่โหลดมาแบบ asynchronous ตอนเริ่มโปรแกรม
แล้วต้องแจกจ่ายให้หลาย module ใช้งานพร้อมกัน

---

## สรุปท้ายบท

ใน Part นี้ซึ่งเป็น Part ที่ 3 ของ Module G เราได้:

- เข้าใจปัญหาที่ `std::future`/`std::async` แก้: การรันงานแบบ asynchronous แล้วรอผล
  ทีหลัง โดยไม่ต้องจัดการ thread, ตัวแปรร่วม, และ mutex ด้วยมือทั้งหมดแบบที่ Part 81-82
  สอนไว้
- ใช้ `std::async` พร้อม Launch Policy (`std::launch::async` vs `std::launch::deferred`)
  ได้อย่างถูกต้อง และเข้าใจอันตรายของการไม่ระบุ policy ให้ชัดเจน
- ใช้ `std::future::get()`/`.wait()`/`.wait_for()` และรู้ข้อจำกัดสำคัญที่สุด: **`.get()`
  เรียกได้เพียงครั้งเดียว**
- ใช้ `std::promise` ส่งค่าข้าม thread แบบ manual เมื่อต้องการควบคุมมากกว่าที่
  `std::async` ให้ได้ และรู้จัก `std::packaged_task` ที่อยู่ตรงกลางระหว่างทั้งสอง
- เข้าใจอย่างลึกซึ้งว่า **exception ที่เกิดใน asynchronous task ถูกเก็บไว้ในสถานะภายใน
  ของ future แล้ว rethrow ตอน `.get()`** ซึ่งแก้ข้อจำกัดของ `std::thread` ธรรมดาที่
  Part 81 หัวข้อ 81.5 พิสูจน์ไปแล้วว่าทำไม่ได้
- เขียนโปรแกรมจริงที่แบ่งงานคำนวณหนัก (นับจำนวนเฉพาะ) ออกเป็นหลายส่วนทำงานแบบขนาน
  แล้วรวมผลลัพธ์ พร้อมเรียนรู้ข้อจำกัดของ Static Partitioning เรื่อง Load Balancing
  ที่จะกลับมาเจออีกครั้งใน Part 85 และ 101

Module G ที่เราเรียนมาตั้งแต่ Part 81 (`std::thread`) ผ่าน Part 82 (`std::mutex`/
`std::atomic`) จนถึง Part นี้ (`std::future`/`std::async`/`std::promise`) ได้ปูพื้นฐาน
เครื่องมือ concurrency หลักของ Modern C++ ไว้ครบถ้วนแล้ว — ตั้งแต่ระดับต่ำที่สุด
(thread ดิบๆ) ไปจนถึงระดับสูงที่สุด (asynchronous task ที่ห่อความซับซ้อนไว้เกือบทั้งหมด)
ใน **Part 84** เราจะเจาะลึกเข้าไปในทิศทางตรงข้าม — กลับไปสำรวจว่าเบื้องหลัง
`std::atomic` ที่เรียนใน Part 82 นั้น สามารถนำไปสร้าง **Data Structure แบบ Lock-Free**
เต็มรูปแบบได้อย่างไร (Lock-Free Programming) ซึ่งเป็นเทคนิคขั้นสูงที่ใช้ในระบบที่ต้องการ
ประสิทธิภาพสูงสุดและมีความหน่วงต่ำที่สุดเท่าที่จะเป็นไปได้

**ต่อไป:** [Part 84 — Lock-Free Programming เบื้องต้น](./part-084-lock-free.md)
