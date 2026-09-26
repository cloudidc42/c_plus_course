# Part 75: C++20 Coroutines (Step 593–600)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 75 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 593–600
> Part ก่อนหน้า: [Part 74 — C++20 Ranges](./part-074-cpp20-ranges.md) | Part ถัดไป: [Part 76 — C++20 Modules](./part-076-cpp20-modules.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า coroutine คืออะไร และต่างจากฟังก์ชันปกติอย่างไรในระดับกลไกการทำงาน
   (stack frame ปกติ vs. coroutine frame ที่ suspend/resume ได้)
2. บอกได้ว่า keyword ใหม่ทั้งสามตัวคือ `co_await`, `co_yield`, `co_return` ทำหน้าที่อะไร
   และรู้กฎว่าฟังก์ชันแบบไหนจะกลายเป็น coroutine โดยอัตโนมัติ
3. เข้าใจว่า compiler แปลง coroutine เป็นโค้ดจริงอย่างไรผ่านกลไก `promise_type` และ
   `std::coroutine_handle`
4. เขียน generator อย่างง่ายด้วย `co_yield` ได้เองตั้งแต่ต้นจนจบ พร้อมเขียน `promise_type`
   เองได้ทุกส่วน
5. เข้าใจ Awaiter Protocol (`await_ready` / `await_suspend` / `await_resume`) ที่อยู่เบื้องหลัง
   `co_await` และเขียนตัวอย่างง่ายๆ ที่ใช้ `co_return` คืนค่าได้
6. อธิบายได้ว่าทำไม C++20 coroutine ถึงถูกเรียกว่าเป็นแค่ "low-level building block"
   ต่างจาก `async`/`await` ใน JavaScript หรือ Python ที่ใช้งานง่ายกว่าเพราะมี runtime มาให้
7. ระบุ use case จริงที่ coroutine เหมาะสม เช่น lazy sequence และงาน async I/O ในอนาคต
8. รู้จักไลบรารีเสริมที่ทำให้ทำงานกับ coroutine ง่ายขึ้นในงานจริง เช่น cppcoro โดยไม่ต้องเขียน
   `promise_type` เองทุกครั้ง

---

## 75.1 Coroutine คืออะไร ต่างจากฟังก์ชันปกติอย่างไร (Step 593)

ก่อนหน้านี้ทุกฟังก์ชันที่เราเขียนใน C/C++ ทำงานแบบเดียวกันหมด: เมื่อถูกเรียก ฟังก์ชันจะได้
**stack frame** ของตัวเอง ทำงานจากบรรทัดแรกไปจนจบ (หรือจนกว่าจะ `return`) แล้ว stack frame
นั้นก็ถูกทำลายทิ้งทันที ฟังก์ชันไม่มีทาง "หยุดกลางคัน" แล้วกลับมาทำต่อจากจุดเดิมได้ — นี่คือ
ธรรมชาติของฟังก์ชันมาตั้งแต่ Part 6 ที่เราเรียนกันมา

**Coroutine** เปลี่ยนกฎนี้ มันคือฟังก์ชันพิเศษที่ **suspend (พักการทำงานกลางคัน) และ resume
(ทำงานต่อจากจุดที่พักไว้) ได้** โดยที่ค่าตัวแปร local ทั้งหมดยังอยู่ครบ ไม่หายไปไหน

```
ฟังก์ชันปกติ:
   เรียก ──▶ ทำงานบรรทัดที่ 1,2,3,...,N ──▶ return ──▶ stack frame ถูกทำลาย

Coroutine:
   เรียก ──▶ ทำงานถึงจุด suspend ──▶ [พัก] ──▶ ผู้เรียกทำงานอย่างอื่นต่อ
                                        │
                     กลับมา resume ◀───┘
                       ทำงานต่อจากจุดเดิม ──▶ [พักอีกครั้ง หรือจบ]
```

เพื่อให้ทำแบบนี้ได้ coroutine ต้องมี "ที่เก็บสถานะ" ที่ไม่ใช่ stack ปกติ (เพราะ stack frame
ถูกคืนทันทีที่ฟังก์ชันคืนการควบคุมกลับไปให้ผู้เรียก) C++ แก้ปัญหานี้ด้วยการสร้าง
**coroutine frame** ซึ่งโดยทั่วไปจะถูกจองบน **heap** (แม้ compiler จะพยายาม optimize ให้อยู่บน
stack ได้ในบางกรณีที่พิสูจน์ได้ว่าปลอดภัย เรียกว่า HALO — Heap Allocation eLision Optimization)
coroutine frame นี้เก็บทุกอย่างที่ต้องรอด: ตัวแปร local, พารามิเตอร์, และ "ตำแหน่งที่ทำงานค้างอยู่"

### coroutine ≠ thread

ข้อควรระวังสำคัญที่สุดสำหรับมือใหม่คือ **coroutine ไม่ใช่ thread** และไม่เกี่ยวกับ
multithreading โดยตรง (เรื่อง thread จริงจะเรียนใน Module G เริ่มที่ Part 81)

| | Thread | Coroutine |
|---|---|---|
| ใครเป็นคนสลับงาน | OS Scheduler (แย่งกัน preemptive) | โค้ดเราเอง เรียก resume() เอง (cooperative) |
| ใช้ CPU core กี่ตัว | สามารถรันขนานจริงบนหลาย core | รันบน thread เดียวเสมอ (เว้นแต่จะย้ายข้าม thread เอง) |
| ต้นทุนการสร้าง | สูง (OS resource, stack เต็มขนาด) | ต่ำมาก (coroutine frame เล็กกว่ามาก) |
| ปัญหา Race Condition | มีถ้าแชร์ข้อมูล | ไม่มี เพราะทำงานทีละจุดตามลำดับที่เราสั่ง resume |

พูดง่ายๆ coroutine คือกลไกใน "ภาษา" ที่ทำให้ฟังก์ชัน **พักได้** ส่วนจะเอาไปใช้ทำ concurrency
จริงหรือไม่นั้นเป็นเรื่องของ library/โครงสร้างที่เราสร้างขึ้นมาครอบมันอีกที (ซึ่งจะพูดถึงใน 75.6)

---

## 75.2 co_await, co_yield, co_return: สามคำสั่งที่ทำให้ฟังก์ชันกลายเป็น Coroutine (Step 594)

C++20 เพิ่ม keyword ใหม่ 3 ตัว และมีกฎง่ายๆ ข้อเดียวคือ **ถ้าฟังก์ชันไหนมี keyword เหล่านี้
ปรากฏอยู่ใน body ของมัน ฟังก์ชันนั้นจะกลายเป็น coroutine โดยอัตโนมัติทันที** (compiler ตรวจจับเอง
ไม่ต้องประกาศอะไรพิเศษ ไม่มี keyword `coroutine` ตรงๆ)

| Keyword | ความหมาย | เทียบใกล้เคียงกับ |
|---|---|---|
| `co_await expr` | พัก coroutine รอจน `expr` (ที่เป็น Awaitable) พร้อม แล้วค่อยทำงานต่อ | `await` ใน JS/Python |
| `co_yield value` | ส่งค่าออกไปให้ผู้เรียก แล้วพักตัวเองไว้ตรงนี้ก่อน (รอ resume ครั้งถัดไป) | `yield` ใน Python generator |
| `co_return [value]` | จบการทำงานของ coroutine (จะมีค่าคืนหรือไม่ก็ได้) | `return` ปกติ แต่ใช้ในบริบท coroutine |

### กฎและข้อจำกัดของ Coroutine

เมื่อฟังก์ชันกลายเป็น coroutine แล้ว มีข้อจำกัดตามมาตรฐานที่ต้องรู้:

1. **ห้ามใช้ `return value;` แบบปกติ** — ต้องใช้ `co_return value;` เท่านั้น (จะใช้ `return;`
   เปล่าๆ เพื่อออกจากฟังก์ชันได้ แต่ใช้คืนค่าไม่ได้)
2. **ค่าที่ coroutine คืนกลับ (return type ของฟังก์ชัน) ต้องเป็น class ที่มี nested type ชื่อ
   `promise_type`** (หรือ compiler หา `coroutine_traits` ที่ตรงกันได้) — จะพูดถึงกลไกนี้ใน 75.3
3. **coroutine ไม่สามารถเป็น**: `main()`, `constexpr` function, constructor, destructor,
   ฟังก์ชันที่ใช้ variadic parameter (`...`) แบบ C-style, หรือฟังก์ชันที่มี `auto` return type
   แบบ deduced ธรรมดา (ต้องใช้ type ที่ประกาศชัดเจนซึ่งมี `promise_type`)
4. `co_await`, `co_yield` ใช้ได้เฉพาะใน body ของ coroutine เท่านั้น (ใช้นอกฟังก์ชันหรือใน lambda
   ที่ไม่ใช่ coroutine จะ error ทันที)

ลองดูตัวอย่างง่ายๆ ที่ยังไม่สมบูรณ์ (จะเติมส่วนที่ขาดใน 75.4) เพื่อดูรูปร่างคร่าวๆ ก่อน:

```cpp
// นี่คือ "รูปร่าง" ของ coroutine — ยังคอมไพล์ไม่ผ่านเพราะ MyGenerator ยังไม่มี promise_type
MyGenerator count_to(int n) {
    for (int i = 1; i <= n; ++i) {
        co_yield i;   // <-- แค่บรรทัดนี้บรรทัดเดียว ก็ทำให้ compiler มองฟังก์ชันนี้เป็น coroutine แล้ว
    }
    co_return;
}
```

จุดที่มือใหม่งงบ่อยที่สุด: **แค่มี `co_yield` โผล่มาสักบรรทัดเดียวในฟังก์ชัน ก็เปลี่ยนความหมาย
ของฟังก์ชันทั้งตัวทันที** ฟังก์ชันจะไม่ "รันจริง" เมื่อถูกเรียกอีกต่อไป แต่จะสร้าง coroutine frame
ขึ้นมาแล้วคืนค่า (object ของ return type) กลับไปทันทีโดยที่โค้ดข้างในยังไม่ได้ทำงานเลยด้วยซ้ำ
(ขึ้นอยู่กับ `initial_suspend` ที่จะอธิบายต่อไป)

---

## 75.3 เบื้องหลัง Compiler: promise_type และ coroutine_handle (Step 595)

นี่คือจุดที่ทำให้ C++20 coroutine ถูกมองว่า "ยาก" กว่าภาษาอื่น เพราะมาตรฐานไม่ได้ให้
generator หรือ task type สำเร็จรูปมาให้ใช้เลย (ต่างจาก Python ที่พิมพ์ `yield` แล้วได้ generator
ใช้งานได้ทันที) — เราต้อง**สร้าง type ที่มี `promise_type` เอง** เพื่อกำหนดพฤติกรรมทุกอย่าง

### ภาพรวมการแปลงโค้ดของ Compiler

เมื่อ compiler เห็นว่าฟังก์ชันเป็น coroutine มันจะแปลง body ของฟังก์ชันเป็นโค้ดประมาณนี้
(นี่คือ pseudocode อธิบายแนวคิด ไม่ใช่โค้ดที่คอมไพล์ได้ตรงๆ):

```cpp
// โค้ดที่เราเขียน:
ReturnType my_coroutine(Args... args) {
    /* body ที่มี co_await / co_yield / co_return */
}

// สิ่งที่ compiler "จินตนาการ" ให้ประมาณนี้ (แนวคิด ไม่ใช่โค้ดจริง):
ReturnType my_coroutine(Args... args) {
    // 1. จองหน่วยความจำสำหรับ coroutine frame (heap หรือ elided)
    allocate coroutine frame {
        promise_type promise;      // สร้าง promise object
        Args... args;              // copy พารามิเตอร์เข้ามาเก็บใน frame
        /* ตัวแปร local ทั้งหมดใน body */
    };

    // 2. เรียก get_return_object() ตั้งแต่ต้น เก็บค่าไว้คืนกลับผู้เรียก
    auto returnObject = promise.get_return_object();

    // 3. co_await promise.initial_suspend();  -- ตัดสินใจว่าจะเริ่มทำงานทันทีหรือพักก่อน

    try {
        /* body จริงของเรา โดยแทนที่:
           co_yield x   -->  co_await promise.yield_value(x);
           co_return x  -->  promise.return_value(x); goto final; (หรือ return_void())
        */
    } catch (...) {
        promise.unhandled_exception();
    }

final:
    // 4. co_await promise.final_suspend();  -- พักก่อนทำลาย frame จริง หรือทำลายเลย
    return returnObject;   // <-- ค่านี้ถูกคืนตั้งแต่ก่อนที่ body จะทำงานเสร็จด้วยซ้ำ!
}
```

จุดสำคัญที่สุดที่ต้องซึมซับ: **`returnObject` ถูกคืนกลับไปให้ผู้เรียกตั้งแต่ช่วงต้น** (หลัง
`initial_suspend`) ไม่ใช่ตอนจบฟังก์ชัน — นี่คือสิ่งที่ทำให้ coroutine "พัก" กลางคันแล้วคืน
ตัวควบคุม (control) กลับไปให้ผู้เรียกได้ตั้งแต่ยังไม่ทำงานอะไรเลยด้วยซ้ำ

### Customization Point ของ promise_type

`promise_type` คือจุดที่เรากำหนด "พฤติกรรม" ของ coroutine ทั้งหมด ต้องมี method ตามรายการนี้
(บาง method จำเป็น บาง method เลือกใส่ตามการใช้งาน):

| Method | จำเป็นหรือไม่ | หน้าที่ |
|---|---|---|
| `get_return_object()` | จำเป็นเสมอ | สร้าง object ที่จะคืนกลับให้ผู้เรียก coroutine (เช่น `Generator<T>`) |
| `initial_suspend()` | จำเป็นเสมอ | ตัดสินว่าเริ่มพักทันที (`suspend_always`) หรือรันเลย (`suspend_never`) |
| `final_suspend() noexcept` | จำเป็นเสมอ | ตัดสินตอนจบว่าจะพักก่อนทำลาย frame หรือไม่ (ต้อง `noexcept`) |
| `unhandled_exception()` | จำเป็นเสมอ | เรียกเมื่อมี exception หลุดออกมาจาก body โดยไม่ถูกจับ |
| `return_void()` | ใส่ถ้าใช้ `co_return;` เปล่าๆ | ทำเมื่อ coroutine จบโดยไม่มีค่าคืน |
| `return_value(T v)` | ใส่ถ้าใช้ `co_return value;` | ทำเมื่อ coroutine จบพร้อมค่าคืน (ใส่ได้แค่ 1 ใน 2 กับ `return_void`) |
| `yield_value(T v)` | ใส่ถ้าใช้ `co_yield` | ทำเมื่อ coroutine ส่งค่าออกมาแบบไม่จบการทำงาน |
| `await_transform(T v)` | ตัวเลือก | ดักแปลง argument ของ `co_await` ก่อนใช้งานจริง (ใช้ทำ scheduler ขั้นสูง) |

### std::suspend_always กับ std::suspend_never

มาตรฐานเตรียม Awaitable สำเร็จรูปมาให้ 2 ตัวใน `<coroutine>` สำหรับกรณีง่ายที่สุด:

```cpp
struct suspend_always {   // พักเสมอ (ไม่ทำอะไร รอ resume() จากภายนอก)
    constexpr bool await_ready() const noexcept { return false; }
    constexpr void await_suspend(std::coroutine_handle<>) const noexcept {}
    constexpr void await_resume() const noexcept {}
};

struct suspend_never {    // ไม่พักเลย (ทำงานต่อทันที)
    constexpr bool await_ready() const noexcept { return true; }
    constexpr void await_suspend(std::coroutine_handle<>) const noexcept {}
    constexpr void await_resume() const noexcept {}
};
```

### std::coroutine_handle<Promise>

นี่คือ "ตัวจับ" (handle) แบบ non-owning ที่ใช้ควบคุม coroutine frame จากภายนอก มี method หลัก:

| Method | หน้าที่ |
|---|---|
| `.resume()` | ทำงานต่อจากจุดที่พักไว้ล่าสุด |
| `.done()` | คืน `true` ถ้า coroutine ทำงานถึง `final_suspend` แล้ว |
| `.destroy()` | ทำลาย coroutine frame (คืนหน่วยความจำ) — ต้องเรียกเองถ้าไม่ resume จนจบ |
| `.promise()` | เข้าถึง `promise_type` object ที่อยู่ใน frame โดยตรง |
| `Handle::from_promise(p)` | (static) สร้าง handle จาก promise object — ใช้ใน `get_return_object()` |

**ข้อสำคัญ**: `coroutine_handle` ไม่ใช่ smart pointer มันไม่ได้จัดการ memory ให้อัตโนมัติ
เราต้องเรียก `.destroy()` เองที่ใดที่หนึ่ง (ปกติคือใน destructor ของ type ที่ห่อ handle ไว้)
ไม่งั้นจะเกิด memory leak แน่นอน — นี่คือเหตุผลที่ type ที่เราสร้างขึ้น (เช่น `Generator<T>`)
ต้องทำตาม RAII เหมือนที่เรียนใน Part 68

---

## 75.4 เขียน Generator ตัวแรกด้วย co_yield (Step 596)

มาลงมือเขียน generator ที่คอมไพล์และรันได้จริงกัน โค้ดนี้ทดสอบแล้วบนเครื่องจริงด้วย
`g++ 13.3.0 -std=c++20`:

```cpp
// generator.cpp
#include <coroutine>
#include <iostream>
#include <utility>   // std::exchange

template <typename T>
struct Generator {
    // ===== ส่วนที่ 1: promise_type — หัวใจของ coroutine =====
    struct promise_type {
        T current_value;

        Generator get_return_object() {
            // สร้าง handle จาก *this (promise object ที่อยู่ใน coroutine frame)
            // แล้วห่อด้วย Generator เพื่อคืนกลับไปให้ผู้เรียก
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        // เริ่ม "พัก" ทันที ไม่รันโค้ดใน body เลยจนกว่าจะมีคนเรียก resume() ครั้งแรก
        std::suspend_always initial_suspend() { return {}; }

        // จบแล้วก็พักไว้เฉยๆ (ให้ตัว Generator เป็นคนเรียก destroy() เอง)
        std::suspend_always final_suspend() noexcept { return {}; }

        // ทุกครั้งที่เจอ co_yield value ใน body จะมาเรียก method นี้
        std::suspend_always yield_value(T value) {
            current_value = value;   // เก็บค่าไว้ให้ผู้เรียกอ่าน
            return {};                // แล้วพักตัวเองไว้ตรงนี้
        }

        void return_void() {}   // ใช้ co_return; เปล่าๆ จบ loop
        void unhandled_exception() { std::terminate(); }
    };

    // ===== ส่วนที่ 2: ตัว Generator เอง — ห่อ handle แบบ RAII =====
    using handle_type = std::coroutine_handle<promise_type>;
    handle_type handle;

    explicit Generator(handle_type h) : handle(h) {}
    ~Generator() { if (handle) handle.destroy(); }   // สำคัญมาก: คืน memory ของ frame

    // ห้าม copy (มีแค่ handle เดียว ถ้า copy กันจะ destroy() ซ้ำ = double free)
    Generator(const Generator&) = delete;
    Generator& operator=(const Generator&) = delete;

    // ย้ายได้ (ยก ownership ของ handle ไป)
    Generator(Generator&& other) noexcept
        : handle(std::exchange(other.handle, nullptr)) {}

    // ===== ส่วนที่ 3: ฟังก์ชันช่วยให้ผู้เรียกใช้งานง่ายขึ้น =====
    bool move_next() {
        handle.resume();          // ทำงานต่อจนกว่าจะเจอ co_yield ครั้งถัดไป หรือจบ
        return !handle.done();
    }

    T current_value() const { return handle.promise().current_value; }
};

// ===== ส่วนที่ 4: coroutine ตัวจริงที่ผู้ใช้มองเห็น — สั้นและอ่านง่ายมาก =====
Generator<int> counter(int start, int end) {
    for (int i = start; i <= end; ++i) {
        co_yield i;
    }
}

int main() {
    auto gen = counter(1, 5);
    while (gen.move_next()) {
        std::cout << gen.current_value() << " ";
    }
    std::cout << std::endl;
}
```

```bash
g++ -Wall -Wextra -std=c++20 generator.cpp -o generator
./generator
```

ผลลัพธ์:

```
1 2 3 4 5
```

### อธิบายว่าเกิดอะไรขึ้นจริงตอนรัน

1. `counter(1, 5)` ถูกเรียก — compiler สร้าง coroutine frame, เรียก `get_return_object()`
   ทันที ได้ `Generator` object กลับมาเก็บใน `gen` **แต่ body ของ `counter` ยังไม่ได้ทำงาน
   เลยสักบรรทัด** เพราะ `initial_suspend()` คืน `suspend_always` (พักตั้งแต่ก่อนเริ่ม)
2. `gen.move_next()` เรียก `handle.resume()` — ตอนนี้เองที่ body เริ่มทำงานจริง วนลูปจน
   `i = 1` แล้วเจอ `co_yield 1` ซึ่งถูกแปลงเป็นการเรียก `promise.yield_value(1)` — เก็บ
   `current_value = 1` แล้วพักตัวเอง (คืน `suspend_always`) ทำให้ `resume()` return กลับมา
3. `gen.current_value()` อ่านค่าที่เก็บไว้ใน promise ผ่าน `handle.promise().current_value`
4. วนซ้ำแบบนี้ไปเรื่อยๆ จนลูปใน `counter` จบ (`i` เกิน 5) แล้ว compiler แทรก
   `promise.return_void()` ให้อัตโนมัติ แล้วไป `final_suspend()` — `handle.done()` จะเป็น
   `true` ทำให้ `move_next()` คืน `false` และ loop ใน `main` หยุด
5. เมื่อ `gen` ออกจาก scope destructor ของ `Generator` เรียก `handle.destroy()` คืน
   หน่วยความจำของ coroutine frame

**generator แบบนี้เรียกว่า "lazy"** เพราะค่าถัดไปจะไม่ถูกคำนวณจนกว่าจะมีคนเรียก
`move_next()` จริงๆ — ต่างจากการสร้าง `std::vector<int>` ที่คำนวณค่าทั้งหมดทันทีตั้งแต่ต้น
ซึ่งเห็นประโยชน์ชัดเจนที่สุดเมื่อ sequence นั้น**ไม่มีที่สิ้นสุด**:

```cpp
// generator ไม่รู้จบ (infinite lazy sequence) — จุดที่ vector ทำไม่ได้เลย
// เพราะ vector ต้องรู้ขนาดล่วงหน้าและคำนวณ "ทุก" ค่าเก็บไว้ในหน่วยความจำ
Generator<long long> fibonacci() {
    long long a = 0, b = 1;
    while (true) {                // <-- true! ไม่มีวันจบ
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

int main() {
    auto gen = fibonacci();
    for (int i = 0; i < 10 && gen.move_next(); ++i) {   // ดึงมาแค่ 10 ตัวแรกเท่าที่ต้องการ
        std::cout << gen.current_value() << " ";
    }
    std::cout << std::endl;
}
```

ทดสอบแล้วให้ผลลัพธ์:

```
0 1 1 2 3 5 8 13 21 34
```

โปรแกรมไม่ค้าง ไม่ overflow หน่วยความจำ เพราะมันคำนวณ "ทีละตัวตามที่ถูกขอ" เท่านั้น — นี่คือ
พลังที่แท้จริงของ lazy evaluation ที่ coroutine มอบให้

---

## 75.5 co_await และ co_return: เจาะลึก Awaiter Protocol (Step 597)

`co_yield x` ที่เราเห็นใน 75.4 จริงๆ แล้วถูกแปลงภายในเป็น `co_await promise.yield_value(x)`
ดังนั้นหัวใจจริงๆ ของ coroutine ทั้งหมดคือ `co_await` และสิ่งที่มันคาดหวังเรียกว่า
**Awaiter** — object ที่มี 3 method ตามนี้:

```cpp
struct MyAwaiter {
    bool await_ready() const noexcept;
    // คืน true = ไม่ต้อง suspend เลย (ทำงานต่อทันที)
    // คืน false = จะ suspend แล้วไปเรียก await_suspend() ต่อ

    void await_suspend(std::coroutine_handle<> h) const noexcept;
    // (หรือคืน bool / คืน coroutine_handle<> อื่น ก็ได้ตามมาตรฐาน — แบบขั้นสูงเรียกว่า
    //  "symmetric transfer" ใช้ทำ scheduler ที่มีประสิทธิภาพสูง ไม่ทำให้ stack ลึกขึ้นเรื่อยๆ)
    // ถูกเรียกทันทีหลัง suspend จริง ได้ handle ของ coroutine ที่กำลังพัก
    // ใช้เก็บ h ไว้ที่ไหนสักที่เพื่อเรียก h.resume() ในภายหลัง (เช่น เมื่อ I/O เสร็จ)

    T await_resume() const noexcept;
    // ถูกเรียกตอน resume กลับมา ค่าที่ return จาก method นี้คือ "ค่าที่ co_await คืนออกมา"
};
```

ลองเขียนตัวอย่างที่รวม `co_await` และ `co_return` (คืนค่า) เข้าด้วยกัน โค้ดนี้ทดสอบแล้ว
บน g++ 13.3.0 เช่นกัน:

```cpp
// task_demo.cpp
#include <coroutine>
#include <iostream>

// Awaiter แบบง่ายที่สุดเท่าที่จะเป็นไปได้: ไม่ suspend จริง แค่โชว์ protocol ของ co_await
struct SyncValue {
    int value;

    bool await_ready() const noexcept { return true; }   // พร้อมอยู่แล้ว ไม่ต้อง suspend
    void await_suspend(std::coroutine_handle<>) const noexcept {}
    int await_resume() const noexcept { return value; }  // ค่าที่ co_await คืนกลับมา
};

struct Task {
    struct promise_type {
        int result = 0;

        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_never initial_suspend() { return {}; }   // รันทันที ไม่ต้องรอ resume ครั้งแรก
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_value(int v) { result = v; }              // รองรับ co_return <ค่า>;
        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle;
    explicit Task(std::coroutine_handle<promise_type> h) : handle(h) {}
    ~Task() { if (handle) handle.destroy(); }

    int result() const { return handle.promise().result; }
};

Task compute() {
    int a = co_await SyncValue{10};   // await_resume() คืน 10 -> a = 10
    int b = co_await SyncValue{32};   // await_resume() คืน 32 -> b = 32
    co_return a + b;                  // เรียก promise.return_value(42)
}

int main() {
    Task t = compute();
    std::cout << "result = " << t.result() << "\n";
}
```

```bash
g++ -Wall -Wextra -std=c++20 task_demo.cpp -o task_demo
./task_demo
# result = 42
```

สังเกตว่า `SyncValue` ในตัวอย่างนี้ **ไม่เคย suspend จริง** (`await_ready()` คืน `true`
เสมอ) มันแค่ทำหน้าที่โชว์ pipeline ของ `co_await`: ส่งค่าเข้าไป → `await_resume()` แปลงเป็น
ค่าที่ expression `co_await SyncValue{10}` ประเมินออกมา ในสถานการณ์จริง (เช่น รอ network
response) `await_ready()` จะคืน `false` และ `await_suspend()` จะเก็บ `coroutine_handle`
ไว้เรียก `.resume()` ในภายหลัง เมื่อข้อมูลจริงพร้อมแล้ว (เช่น จาก callback ของ OS หรือ
event loop) — นี่คือกลไกที่ทำให้ "async I/O" เป็นไปได้ ซึ่งจะพูดถึงเพิ่มใน 75.7

### ค่าที่ await_suspend คืนได้: void, bool, และ coroutine_handle<> อื่น

`await_suspend()` ไม่จำเป็นต้องคืน `void` เสมอไปตามที่เห็นใน `std::suspend_always` มาตรฐาน
อนุญาตให้คืนได้ 3 แบบ ซึ่งแต่ละแบบมีความหมายต่างกัน:

| Return Type | ความหมาย |
|---|---|
| `void` | suspend แน่นอน ไม่มีทางยกเลิก |
| `bool` | คืน `true` = suspend จริง / คืน `false` = **ยกเลิกการ suspend** ทำงานต่อทันที |
| `std::coroutine_handle<>` (ตัวอื่น) | suspend ตัวเอง แล้ว**เรียก resume ของ coroutine อีกตัวที่คืนมาต่อทันที** (เทคนิคนี้เรียกว่า **symmetric transfer** ใช้ทำ scheduler ที่ส่งต่องานกันหลายชั้นโดยไม่ทำให้ stack ลึกขึ้นเรื่อยๆ ทีละชั้น) |

ลองดูตัวอย่างแบบ `bool` ที่ทดสอบคอมไพล์และรันจริงแล้ว:

```cpp
// Awaiter ที่ await_suspend คืน bool: true = suspend จริง, false = ทำงานต่อทันที
struct ConditionalAwait {
    bool should_suspend;
    bool await_ready() const noexcept { return false; }
    bool await_suspend(std::coroutine_handle<>) const noexcept {
        std::cout << "[await_suspend called] ";
        return should_suspend;   // false -> ยกเลิก suspend ทำงานต่อทันที
    }
    void await_resume() const noexcept {}
};

Task demo() {
    std::cout << "before\n";
    co_await ConditionalAwait{false};   // false -> ไม่ suspend จริง ทำงานต่อทันที
    std::cout << "after\n";
}
```

ทดสอบจริงได้ผลลัพธ์:

```
before
[await_suspend called] after
```

สังเกตว่า `await_suspend()` **ถูกเรียกจริง** (เพราะ `await_ready()` คืน `false`) แต่เพราะ
มันคืน `false` กลับมา coroutine เลย**ไม่ suspend จริง** ทำงานต่อจนจบทันทีในการเรียกครั้งแรก
โดยไม่ต้องมีใครมาเรียก `.resume()` เพิ่ม — เทคนิคนี้มีประโยชน์เมื่ออยากเช็คเงื่อนไขบางอย่าง
ก่อนตัดสินใจว่าจะ suspend จริงหรือไม่ (เช่น เช็คว่าข้อมูลพร้อมอยู่แล้วหรือยัง ถ้าพร้อมแล้ว
ก็ไม่จำเป็นต้อง suspend ให้เสียเวลา)

---

## 75.6 ทำไม C++20 Coroutine เป็นแค่ "Low-Level Building Block" (Step 598)

ถ้าเทียบกับ `async`/`await` ใน JavaScript หรือ Python จะพบว่า C++20 "ยากกว่ามาก" ในการ
เริ่มต้นใช้งาน ทั้งที่ keyword หน้าตาคล้ายกัน เหตุผลคือ **มาตรฐาน C++20 ให้แค่ไวยากรณ์และ
"โปรโตคอล" ว่า compiler ควรแปลงโค้ดอย่างไร แต่ไม่ได้แถม runtime, scheduler, หรือ event loop
มาให้เลย**

| | JavaScript / Python `async`/`await` | C++20 Coroutine |
|---|---|---|
| Generator/Task type สำเร็จรูป | มี (`generator`, `Promise`, `Future`) | **ไม่มี** — ต้องเขียน `promise_type` เอง |
| Event loop / scheduler | มีมากับ runtime (V8, asyncio) | **ไม่มี** — ต้องหาไลบรารีเสริมหรือเขียนเอง |
| การจัดการ error | try/catch ทำงานข้าม await ได้เลย | ต้องออกแบบเองผ่าน `unhandled_exception()` |
| หน่วยความจำ | Garbage Collector จัดการให้ | ต้องจัดการเอง (`coroutine_handle::destroy()`) |
| ความยืดหยุ่น | ถูกจำกัดด้วยดีไซน์ของ runtime | ปรับแต่งได้ทุกจุด (allocator เอง, scheduler เอง) |

พูดง่ายๆ ทีมออกแบบ C++20 (Core Language) ตั้งใจสร้างแค่ **"เครื่องมือระดับต่ำ (mechanism)"**
ที่ทรงพลังและยืดหยุ่นที่สุดเท่าที่จะทำได้ แล้วปล่อยให้ **library ระดับสูงกว่า** (เช่น
คนเขียน framework, ไลบรารีเครือข่าย) เอาไปสร้าง generator, task, async I/O ที่ใช้งานง่าย
ขึ้นมาอีกที — นี่คือปรัชญาการออกแบบของ C++ มาตลอด (zero-overhead abstraction, "pay only
for what you use") ต่างจาก JS/Python ที่ตัดสินใจแทนผู้ใช้ไปเลยว่าจะให้ runtime หน้าตาแบบไหน

ข้อดีของแนวทางนี้: เราสามารถเขียน coroutine ที่ **ไม่จอง heap เลย** (ผ่าน HALO), ทำ
scheduler ของตัวเองที่ optimize เฉพาะงาน (เช่น game engine, embedded system ที่ไม่มี heap
ให้ใช้อิสระ) ได้ — สิ่งที่ทำไม่ได้เลยใน JS/Python เพราะ runtime ถูกล็อกไว้ตายตัว

ข้อเสีย: การเริ่มต้นใช้งานยากกว่ามาก ต้องเข้าใจ `promise_type` ก่อนจะเขียน generator
ง่ายๆ ได้สักตัว นี่คือเหตุผลที่โปรเจกต์จริงในอุตสาหกรรมมักไม่เขียน `promise_type` เองจาก
ศูนย์ แต่ใช้ไลบรารีที่ทำสิ่งนี้ให้แล้ว (พูดถึงใน 75.8)

---

## 75.7 Use Case จริงที่ Coroutine เหมาะสม (Step 599)

### 1. Lazy Sequence / Infinite Generator

ตามที่เห็นใน 75.4 — เหมาะกับข้อมูลที่มีขนาดใหญ่มากหรือไม่รู้จบ, ข้อมูลที่คำนวณแพงและอยาก
คำนวณเฉพาะส่วนที่ถูกใช้จริง (เช่น อ่านไฟล์ทีละบรรทัดแบบ lazy, สร้าง permutation ทีละตัว,
เดิน traverse โครงสร้างข้อมูลแบบ tree/graph ทีละ node โดยไม่ต้องสร้าง list ทั้งหมดไว้ล่วงหน้า)

### 2. Parser และ State Machine

โค้ดที่ต้อง "หยุดรอข้อมูลเพิ่ม" กลางคันแล้วทำงานต่อ (เช่น parser ที่อ่านข้อมูลมาทีละ chunk
จาก network ไม่ครบประโยคในคราวเดียว) coroutine ทำให้เขียน logic การ parse เป็นลำดับขั้นตอน
ที่อ่านง่ายราวกับเขียนแบบ synchronous ทั้งที่จริงๆ มันพักรอข้อมูลอยู่เบื้องหลัง

### 3. Async I/O ในอนาคต (แนวคิด)

นี่คือ use case ที่คนพูดถึงมากที่สุดเวลานึกถึง coroutine: การเขียนโค้ดเครือข่าย/ไฟล์แบบ
asynchronous แต่ให้**อ่านเหมือนโค้ด synchronous ธรรมดา** ไม่ต้องซ้อน callback หลายชั้น
(เรียกปัญหานี้ว่า "callback hell")

```cpp
// ภาพจำลอง "อนาคตที่อยากไปให้ถึง" ด้วย library ระดับสูง (เช่น cppcoro หรือคล้ายกัน)
// นี่คือ pseudocode เพื่ออธิบายแนวคิด ไม่ใช่โค้ด C++20 มาตรฐานเปล่าๆ ที่คอมไพล์ได้ทันที
Task<std::string> fetch_and_process(std::string url) {
    std::string response = co_await http_get_async(url);   // พักรอ network โดยไม่บล็อก thread
    std::string parsed   = co_await parse_async(response); // พักรอ parsing (ถ้าทำ async)
    co_return parsed;
}
```

โค้ดหน้าตาเหมือน synchronous 100% แต่ **thread จริงไม่ได้ถูกบล็อกรอ** ระหว่างที่รอ network
เลย — thread นั้นถูกปล่อยกลับไปทำงานอื่น (เช่น รับ request ใหม่) และจะถูกเรียก `.resume()`
กลับมาทำงานต่อเมื่อข้อมูลจาก network มาถึงจริงๆ ผ่านกลไก `await_suspend()` ที่ผูกกับ
event loop ของระบบ (เช่น `epoll` บน Linux ที่เราเรียนใน Part 34) นี่คือรูปแบบที่ web
framework และ database driver รุ่นใหม่จำนวนมากในโลก C++ กำลังปรับมาใช้ เพราะรองรับ
connection จำนวนมากพร้อมกันโดยใช้ thread น้อยกว่าการสร้าง thread ต่อ connection แบบเดิม
มาก (หัวข้อนี้จะกลับมาเจาะลึกอีกครั้งเมื่อเรียน Web Development ใน Module I)

**ข้อควรระวังตามหลักความซื่อสัตย์ของหลักสูตรนี้**: โค้ด `http_get_async` ในตัวอย่างข้างบน
**ไม่มีอยู่จริงใน C++20 มาตรฐาน** มันคือสิ่งที่ต้องมาจากไลบรารีภายนอกที่เขียน Awaiter
ผูกกับ network stack ของ OS เอง มาตรฐานให้แค่ "ไวยากรณ์" แต่ไม่ได้ให้ "การเชื่อมต่อกับ
world จริง" มาด้วย

---

## 75.8 ไลบรารีเสริมที่ทำให้ Coroutine ใช้ง่ายขึ้น (Step 600)

เพราะการเขียน `promise_type` เองทุกครั้งนั้นน่าเบื่อและเสี่ยงบั๊ก (ลืม `destroy()`, ลืม
`return_void()`, เขียน Awaiter ผิด) ในงานจริงแทบไม่มีใครเขียน coroutine type ตั้งแต่ศูนย์
ทุกโปรเจกต์ — จะใช้ไลบรารีที่ทำสิ่งเหล่านี้ให้สำเร็จรูปแทน ในระดับ "เกริ่นให้รู้จัก"
(ไม่ลงรายละเอียดการติดตั้ง/ใช้งานในบทนี้ เพราะเป็นหัวข้อใหญ่พอจะเป็นบทเรียนของตัวเอง):

| ไลบรารี | จุดเด่น |
|---|---|
| **cppcoro** (โดย Lewis Baker) | ไลบรารี coroutine ยุคแรกๆ ที่ได้รับความนิยมสูง มี `generator<T>`, `task<T>`, `async_generator<T>`, `sync_wait()`, `when_all()` พร้อมใช้ ไม่ต้องเขียน `promise_type` เอง |
| **Boost.Cobalt** | ไลบรารีใน Boost (เพิ่มเข้ามาในเวอร์ชันหลังๆ) ให้ `task`, `generator`, `channel` แบบ coroutine พร้อม I/O integration กับ Boost.Asio |
| **stdexec / P2300 "Senders and Receivers"** | ข้อเสนอสำหรับมาตรฐาน C++ ในอนาคต (ยังไม่ใช่ส่วนหนึ่งของ C++20/23) ที่พยายามสร้างมาตรฐานกลางสำหรับ async operation ที่ทำงานร่วมกับ coroutine ได้ดี — ถือเป็นทิศทางที่คณะกรรมการมาตรฐานกำลังผลักดันสำหรับอนาคตของ async C++ |
| **libcoro**, **asio (ผ่าน `awaitable<T>`)** | Boost.Asio และ standalone Asio รุ่นใหม่รองรับการเขียน network code แบบ `co_await` ได้โดยตรงแล้ว เป็นตัวอย่างจริงของ use case ใน 75.7 |

แนวทางที่แนะนำสำหรับผู้เรียน: **เข้าใจกลไกเบื้องหลัง (`promise_type`, Awaiter Protocol)
ให้แน่นตามที่เรียนใน Part นี้ก่อน** เพราะจะช่วยให้อ่านและ debug โค้ดที่ใช้ไลบรารีเหล่านี้
ได้เข้าใจจริง ไม่ใช่แค่ก็อปโค้ดมาใช้โดยไม่รู้ว่าเบื้องหลังเกิดอะไรขึ้น — เมื่อเข้าใจกลไกแล้ว
การไปหยิบไลบรารีสำเร็จรูปมาใช้ในงานจริงจะง่ายขึ้นมาก เพราะรู้ว่า `task<T>` หรือ
`generator<T>` ของไลบรารีเหล่านั้น "ทำอะไรอยู่ข้างใน" ไม่ใช่กล่องดำที่มองไม่ทะลุ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมเรียก `handle.destroy()`** — ทำให้ coroutine frame ที่จองบน heap ไม่ถูกคืน กลายเป็น
   memory leak ทุกครั้งที่เรียก coroutine แล้วไม่จัดการ handle ให้ดี วิธีป้องกันที่ปลอดภัย
   ที่สุดคือห่อ handle ด้วย RAII object เสมอ (แบบ `Generator`/`Task` ในบทนี้) อย่าเก็บ
   `coroutine_handle` แบบดิบๆ ไว้ในโค้ดโดยไม่มีใครรับผิดชอบทำลายมัน

2. **ลืมลบ copy constructor ของ type ที่ห่อ handle** — ถ้า `Generator` copy ได้ตามปกติ
   (compiler generate copy constructor ให้ฟรี) จะมี 2 object ที่ถือ `coroutine_handle`
   ตัวเดียวกัน พอทั้งคู่ออกจาก scope destructor จะเรียก `.destroy()` ซ้ำสองครั้ง เป็น
   **double free** ทันที ต้อง `= delete` copy constructor/assignment เสมอ (หรือทำ handle
   แบบนับ reference เองถ้าอยากให้ copy ได้จริงๆ ซึ่งซับซ้อนกว่ามาก)

3. **ใช้ `co_return value;` แต่ `promise_type` ไม่มี `return_value()`** — จะได้ error
   ตอนคอมไพล์ทันที ตัวอย่าง error message จริงจาก g++ 13.3.0:
   ```
   error: no member named 'return_value' in '...promise_type' {aka 'BadTask::promise_type'}
   ```
   จำไว้ว่า `return_void()` กับ `return_value(T)` ใส่ได้แค่อย่างใดอย่างหนึ่งใน
   `promise_type` เดียวกัน (ใส่ทั้งคู่ก็ error เพราะ compiler ไม่รู้จะเลือกอันไหน)

4. **แก้ `promise_type` แต่ compiler ยัง error หา constructor ของ return type ไม่เจอ** —
   คนที่เพิ่งเริ่มมักลืมว่า `get_return_object()` ต้องคืนค่าที่สร้างจาก
   `coroutine_handle<promise_type>::from_promise(*this)` ให้ถูกต้อง ไม่ใช่สร้าง
   `Generator` ด้วย default constructor เฉยๆ (ซึ่งจะไม่มี handle ผูกกับ frame จริงเลย)

5. **capture argument แบบ reference เข้า coroutine แล้วปล่อยให้ตัวแปรต้นทางหมดอายุ** —
   นี่คือบั๊กที่พบบ่อยที่สุดในโค้ด coroutine จริง เช่น:
   ```cpp
   Generator<int> bad_example(const std::vector<int>& data) { // รับมาโดย reference
       for (int x : data) co_yield x;
   }
   auto gen = bad_example({1, 2, 3});  // temporary vector! ถูกทำลายทันทีหลังบรรทัดนี้จบ
   gen.move_next();  // Undefined Behavior: data ที่ coroutine อ้างถึงตายไปแล้ว
   ```
   เพราะ coroutine "พัก" ข้ามหลาย statement การอ้างอิงไปยัง object ชั่วคราว (temporary)
   หรือ object ที่อายุสั้นกว่า coroutine frame เองจะเป็นอันตรายเสมอ — กฎที่ปลอดภัยคือ
   **รับพารามิเตอร์ที่อาจมีอายุสั้นเข้ามาโดย value (copy) เข้าไปเก็บไว้ใน frame แทนการรับ
   โดย reference** เว้นแต่จะมั่นใจจริงๆ ว่า object ต้นทางจะมีอายุยืนยาวกว่า coroutine

6. **ลืมว่า body ของ coroutine ไม่ได้ทำงานทันทีที่เรียก** ถ้า `initial_suspend()` คืน
   `suspend_always` — มือใหม่มักคาดหวังว่าเรียก `counter(1, 5)` แล้วโค้ดข้างในทำงานทันที
   ทั้งที่จริงต้องเรียก `move_next()`/`resume()` ครั้งแรกก่อนถึงจะเริ่มทำงานจริง

7. **สับสนระหว่าง coroutine กับ multithreading** ตามที่อธิบายใน 75.1 — coroutine เดียว
   ไม่ได้ทำให้โค้ดรันขนานกันจริงบนหลาย core มันแค่ทำให้ฟังก์ชัน "พักได้" บน thread เดียว
   ถ้าต้องการใช้หลาย core จริงยังต้องพึ่ง `std::thread` (Module G) ควบคู่กันไป

---

## แบบฝึกหัดท้ายบท

1. เขียน coroutine generator ที่สร้างเลขคู่ (even number) ตั้งแต่ 0 ถึง N (รับ N เป็น
   พารามิเตอร์) โดยใช้โครง `Generator<T>` จาก 75.4

2. ดัดแปลง `Generator<T>` ให้ทำงานกับข้อมูลชนิด `std::string` ได้ แล้วเขียน coroutine ที่
   `co_yield` คำ (word) ทีละคำจากประโยคที่รับเข้ามา (ใบ้: ใช้ `std::istringstream` แยกคำ)

3. เขียนฟังก์ชัน `take(Generator<T>& gen, int n)` ที่รับ generator กับจำนวน `n` แล้วคืน
   `std::vector<T>` ที่มีค่า n ตัวแรกจาก generator นั้น (ประยุกต์จาก fibonacci generator
   ใน 75.4 — ทดสอบด้วย `take(fibonacci(), 15)`)

4. อธิบายด้วยคำพูดของตัวเอง (เขียนเป็นคอมเมนต์ในโค้ด หรือเอกสารสั้นๆ) ว่าทำไมตัวอย่าง
   `bad_example` ในข้อ 5 ของ Common Pitfalls ถึงเป็น Undefined Behavior พร้อมเขียนโค้ด
   แก้ไขให้ปลอดภัย (รับพารามิเตอร์โดย value แทน)

5. เขียน `promise_type` ของ Awaiter/Task ใหม่ที่นับจำนวนครั้งที่ coroutine ถูก resume
   (เก็บ counter ไว้ใน `promise_type` แล้ว print ออกมาตอนจบว่า resume ไปกี่ครั้ง)

6. (ท้าทาย) ลองเขียน `Generator<T>` เวอร์ชันที่รองรับการโยน exception ออกจากภายใน
   coroutine แล้วให้ผู้เรียก (`move_next()`) เป็นคนรับ exception นั้นแทนที่จะเรียก
   `std::terminate()` ใน `unhandled_exception()` (ใบ้: เก็บ `std::exception_ptr` ไว้ใน
   promise ด้วย `std::current_exception()` แล้ว `std::rethrow_exception()` ใน `move_next()`)

### แนวทางเฉลยข้อ 1

```cpp
#include <coroutine>
#include <iostream>
#include <utility>

template <typename T>
struct Generator {
    struct promise_type {
        T current_value;
        Generator get_return_object() {
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(T value) {
            current_value = value;
            return {};
        }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    using handle_type = std::coroutine_handle<promise_type>;
    handle_type handle;
    explicit Generator(handle_type h) : handle(h) {}
    ~Generator() { if (handle) handle.destroy(); }
    Generator(const Generator&) = delete;
    Generator(Generator&& other) noexcept : handle(std::exchange(other.handle, nullptr)) {}
    bool move_next() { handle.resume(); return !handle.done(); }
    T current_value() const { return handle.promise().current_value; }
};

Generator<int> even_numbers(int n) {
    for (int i = 0; i <= n; i += 2) {
        co_yield i;
    }
}

int main() {
    auto gen = even_numbers(10);
    while (gen.move_next()) {
        std::cout << gen.current_value() << " ";
    }
    std::cout << std::endl;   // คาดหวัง: 0 2 4 6 8 10
}
```

ทดสอบคอมไพล์และรันจริง (`g++ -Wall -Wextra -std=c++20`) ได้ผลลัพธ์ตรงตามที่คาดไว้:
`0 2 4 6 8 10`

### แนวทางเฉลยข้อ 3

```cpp
#include <coroutine>
#include <iostream>
#include <utility>
#include <vector>

template <typename T>
struct Generator {
    struct promise_type {
        T current_value;
        Generator get_return_object() {
            return Generator{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(T value) {
            current_value = value;
            return {};
        }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };
    using handle_type = std::coroutine_handle<promise_type>;
    handle_type handle;
    explicit Generator(handle_type h) : handle(h) {}
    ~Generator() { if (handle) handle.destroy(); }
    Generator(const Generator&) = delete;
    Generator(Generator&& other) noexcept : handle(std::exchange(other.handle, nullptr)) {}
    bool move_next() { handle.resume(); return !handle.done(); }
    T current_value() const { return handle.promise().current_value; }
};

Generator<long long> fibonacci() {
    long long a = 0, b = 1;
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

// ฟังก์ชัน take: ดึงค่า n ตัวแรกจาก generator ใดๆ ออกมาเป็น vector
template <typename T>
std::vector<T> take(Generator<T>& gen, int n) {
    std::vector<T> result;
    result.reserve(n);
    for (int i = 0; i < n && gen.move_next(); ++i) {
        result.push_back(gen.current_value());
    }
    return result;
}

int main() {
    auto gen = fibonacci();
    std::vector<long long> first15 = take(gen, 15);
    for (auto v : first15) std::cout << v << " ";
    std::cout << std::endl;
}
```

ทดสอบคอมไพล์และรันจริงได้ผลลัพธ์:
`0 1 1 2 3 5 8 13 21 34 55 89 144 233 377`

ข้อสังเกตสำคัญ: ฟังก์ชัน `take` รับ `Generator<T>&` (reference ธรรมดา ไม่ใช่ `const&`)
เพราะ `move_next()` ต้องเปลี่ยนสถานะภายในของ handle (มัน resume ทำให้ frame เปลี่ยนสถานะ)
และรับ `gen` ที่สร้างไว้แล้วจากภายนอกแทนที่จะสร้าง generator ใหม่ข้างใน `take` เอง
เพื่อให้ `main` ยังคุม lifetime ของ `gen` ได้ต่อ (จะเรียก `take` ซ้ำอีกกี่ครั้งก็ได้กับ
generator ตัวเดิมที่ทำงานต่อจากจุดเดิม)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า coroutine คือฟังก์ชันที่ suspend/resume การทำงานได้ ต่างจากฟังก์ชันปกติที่รัน
  จนจบทีเดียวแล้ว stack frame ถูกทำลายทันที
- รู้จัก keyword ใหม่ทั้งสามตัว `co_await`, `co_yield`, `co_return` และกฎที่ทำให้ฟังก์ชัน
  กลายเป็น coroutine โดยอัตโนมัติ
- เจาะลึกกลไกเบื้องหลังที่ compiler ใช้แปลงโค้ด ผ่าน `promise_type` และ
  `std::coroutine_handle` ครบทุก customization point
- เขียน generator ด้วย `co_yield` ได้จริงตั้งแต่ต้นจนจบ ทั้งแบบจำกัดจำนวนและแบบไม่รู้จบ
  (infinite lazy sequence) และทดสอบคอมไพล์รันจริงบน g++ 13.3.0 ทุกตัวอย่าง
- เข้าใจ Awaiter Protocol (`await_ready`/`await_suspend`/`await_resume`) เบื้องหลัง
  `co_await` ผ่านตัวอย่าง `Task` ที่ใช้ `co_return` คืนค่า
- เข้าใจว่าทำไม C++20 coroutine ถึงเป็นแค่ low-level building block ที่ต้องพึ่งไลบรารี
  เสริมถึงจะใช้งานสะดวกเทียบเท่า `async`/`await` ของ JS/Python
- รู้จัก use case จริงที่ coroutine เหมาะสม (lazy sequence, parser, และแนวทาง async I/O
  ในอนาคต) พร้อมรู้จักไลบรารีเสริมอย่าง cppcoro, Boost.Cobalt เป็นแนวทางต่อยอด

coroutine เป็นหนึ่งในฟีเจอร์ที่ซับซ้อนที่สุดที่ C++20 เพิ่มเข้ามา แต่ก็เป็นรากฐานสำคัญของ
การเขียนโค้ด asynchronous ยุคใหม่ ใน **Part 76** เราจะไปดูอีกหนึ่งฟีเจอร์ใหญ่ของ C++20
ที่พยายามแก้ปัญหาคลาสสิกของภาษา C/C++ มาตั้งแต่ Part 1 — ระบบ `#include` — นั่นคือ
**C++20 Modules**

**ต่อไป:** [Part 76 — C++20 Modules](./part-076-cpp20-modules.md)
