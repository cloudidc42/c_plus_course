# Part 70: Move Semantics และ Rvalue Reference (Step 553–560)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 70 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 553–560
> Part ก่อนหน้า: [Part 69 — ภาพรวม C++11](./part-069-cpp11-overview.md) | Part ถัดไป: [Part 71 — constexpr และ Compile-Time Programming](./part-071-constexpr.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. แยกแยะ **lvalue** และ **rvalue** ได้อย่างถูกต้องในนิพจน์ C++ ทุกรูปแบบ และอธิบายได้ว่าทำไม
   ความแตกต่างนี้ถึงเป็นรากฐานของฟีเจอร์ทั้งหมดใน Part นี้
2. อธิบายปัญหาที่แท้จริงที่ **move semantics** เข้ามาแก้ไข นั่นคือการ copy ข้อมูลจำนวนมากโดย
   ไม่จำเป็นเมื่อส่งต่อหรือ return ทรัพยากรระหว่างฟังก์ชัน
3. ใช้ **rvalue reference** (`&&`) ประกาศ overload ของฟังก์ชันที่แยกแยะระหว่างการรับ lvalue
   กับ rvalue ได้อย่างถูกต้อง
4. เข้าใจอย่างถ่องแท้ว่า **`std::move` เป็นแค่การ cast** ไม่ได้ "ย้าย" อะไรด้วยตัวมันเองเลย
   และรู้ว่าการย้ายที่แท้จริงเกิดขึ้นที่ไหน
5. เขียน **move constructor** และ **move assignment operator** เองสำหรับ class ที่ถือ raw
   pointer ได้อย่างถูกต้องและปลอดภัย (ทบทวนสไตล์ class จาก Part 51)
6. อธิบายและประยุกต์ใช้ **Rule of Five** ที่ขยายจาก Rule of Three เดิม (Part 46, 51, 68) ได้
   อย่างถูกต้องครบถ้วน
7. ใช้ **`std::swap`** และเขียน **copy-and-swap idiom** เพื่อให้ `operator=` รองรับได้ทั้ง copy
   และ move ด้วยฟังก์ชันเดียว พร้อมได้ strong exception guarantee ฟรี (ทบทวน Part 68.7)
8. เข้าใจแนวคิดเบื้องต้นของ **perfect forwarding** ผ่าน `std::forward` และ **forwarding
   reference** (universal reference) เพื่อเขียนฟังก์ชัน wrapper ที่ส่งต่ออาร์กิวเมนต์โดยไม่เสีย
   ข้อมูลว่าเป็น lvalue หรือ rvalue

---

## 70.1 Lvalue กับ Rvalue คืออะไร (Step 553)

ก่อนจะเข้าใจ move semantics ได้อย่างแท้จริง ต้องเข้าใจแนวคิดพื้นฐานที่สุดของภาษา C++ ก่อน นั่นคือ
**value category** (ประเภทของค่า) ซึ่งแบ่งนิพจน์ทุกตัวในภาษา C++ ออกเป็นสองกลุ่มหลัก (ในทาง
เทคนิคมีมากกว่านี้ เช่น `xvalue`, `glvalue`, `prvalue` แต่สำหรับระดับที่ต้องใช้งานจริง สองกลุ่ม
หลักนี้เพียงพอแล้ว):

> **คำอธิบายแบบเข้าใจง่ายที่สุด**:
> - **lvalue** (locator value) คือ **สิ่งที่มีชื่อและมีที่อยู่ในหน่วยความจำจริง** สามารถเอา
>   `&` มาหา address ได้ และยังคง "อยู่" ต่อไปได้หลังจบนิพจน์ปัจจุบัน
> - **rvalue** (right-hand value) คือ **ค่าชั่วคราวที่ไม่มีชื่อของตัวเอง** มักปรากฏแค่ทาง
>   ด้านขวาของเครื่องหมาย `=` และหายไปทันทีที่นิพจน์นั้นจบลง

```cpp
// 01_lvalue_rvalue.cpp
#include <iostream>

int getFive() { return 5; }

int main() {
    int x = 10;          // x เป็น lvalue: มีชื่อ มีที่อยู่ในหน่วยความจำจริง อยู่ได้นานกว่าหนึ่งบรรทัด
    int y = x;            // x ทางขวาของ = คือการ "อ่านค่า" จาก lvalue

    int* px = &x;         // ถูก: เอา address ของ x ได้ เพราะ x เป็น lvalue (มีที่อยู่จริง)
    (void)px;

    // int* bad = &10;    // ผิด! 10 เป็น rvalue (ค่าชั่วคราว ไม่มีที่อยู่ถาวรให้ชี้)
    // int* bad2 = &getFive(); // ผิดเช่นกัน! ค่าที่ return กลับมาเป็น rvalue ชั่วคราว

    int sum = x + y;       // (x + y) เป็นนิพจน์ที่ให้ผลลัพธ์เป็น rvalue: มีอยู่แค่ชั่วคราวระหว่างประมวลผล
    std::cout << "sum = " << sum << '\n';

    // กฎจำง่ายๆ: rvalue คือสิ่งที่ปรากฏ "ทางขวาของเครื่องหมาย =" แล้วไม่มีชื่อของตัวเองอยู่ทางซ้าย
    // - ตัวแปรที่มีชื่อ (x, y)        -> lvalue
    // - ค่าคงที่ตรงๆ (10, 3.14)      -> rvalue
    // - ผลลัพธ์จาก operator (x + y)  -> rvalue
    // - ค่าที่ return จากฟังก์ชันแบบไม่ใช่ reference (getFive()) -> rvalue

    x = getFive();          // x (lvalue) รับค่าจาก getFive() (rvalue) ได้ตามปกติ -- นี่คือ assignment ธรรมดา
    std::cout << "x = " << x << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 01_lvalue_rvalue.cpp -o 01_lr
./01_lr
```

```
sum = 20
x = 5
```

### ตารางสรุป: วิธีแยกแยะ lvalue/rvalue อย่างรวดเร็ว

| นิพจน์ | ประเภท | เหตุผล |
|---|---|---|
| `x` (ตัวแปรที่ประกาศไว้) | lvalue | มีชื่อ มีที่อยู่ในหน่วยความจำ |
| `10`, `3.14`, `'a'` | rvalue | ค่าคงที่ตรงตัวอักษร (literal) ไม่มีที่อยู่ถาวร |
| `x + y`, `a * b` | rvalue | ผลลัพธ์ชั่วคราวจาก operator |
| `getFive()` (คืนค่าแบบไม่ใช่ reference) | rvalue | ค่าที่ return มาเป็นสำเนาชั่วคราว |
| `std::string("hello")` (temporary object) | rvalue | object ชั่วคราวที่ยังไม่ได้ผูกกับชื่อใดๆ |
| `arr[0]` | lvalue | อ้างอิงตำแหน่งจริงใน array |
| `*ptr` (dereference) | lvalue | อ้างอิงตำแหน่งจริงที่ pointer ชี้ไป |

ทำไมเรื่องนี้ถึงสำคัญมากจนต้องมีคำศัพท์เฉพาะ? เพราะ **lvalue กับ rvalue มีนัยสำคัญต่อการจัดการ
ทรัพยากรที่แตกต่างกันโดยสิ้นเชิง**: lvalue คือสิ่งที่ "ยังจะถูกใช้ต่อ" เราจึงต้อง **copy** ข้อมูล
ออกมาถ้าต้องการเก็บสำเนาไว้ใช้ที่อื่น แต่ rvalue คือสิ่งที่ "กำลังจะถูกทิ้งไปอยู่แล้วในไม่ช้า"
(เช่น temporary object ที่จะหมดอายุทันทีที่จบ statement) เราจึง **ขโมยทรัพยากรของมันมาใช้ได้เลย
โดยไม่ต้อง copy** เพราะยังไงเจ้าของเดิมก็จะถูกทำลายอยู่ดี — นี่คือแก่นความคิดทั้งหมดของ move
semantics ที่จะเรียนต่อไปในหัวข้อถัดๆ ไป

---

## 70.2 ปัญหาที่ Move Semantics แก้: การ Copy ข้อมูลจำนวนมากโดยไม่จำเป็น (Step 554)

ก่อน C++11 เมื่อฟังก์ชันต้อง return object ขนาดใหญ่ หรือเมื่อ `std::vector` ต้องขยายขนาด
(reallocate) เพื่อรองรับ element ใหม่ วิธีเดียวที่ compiler รู้จักคือ **deep copy** ทุกครั้ง —
คัดลอกข้อมูลทั้งหมดไปยังหน่วยความจำใหม่ ทั้งที่ในหลายกรณี **ต้นฉบับกำลังจะถูกทำลายทิ้งอยู่แล้ว
ในบรรทัดถัดไป** การ copy จึงเป็นการเสียเวลาและหน่วยความจำโดยสิ้นเชิง

```cpp
// 02_why_move.cpp - ปัญหาที่ move semantics แก้: copy ข้อมูลจำนวนมากโดยไม่จำเป็น
#include <iostream>
#include <utility>
#include <vector>

class BigBuffer {
public:
    explicit BigBuffer(std::size_t n) : size_(n), data_(new int[n]) {
        for (std::size_t i = 0; i < n; ++i) data_[i] = static_cast<int>(i);
        std::cout << "  [สร้างใหม่] จอง " << size_ << " int\n";
    }

    ~BigBuffer() {
        delete[] data_;
    }

    // Copy constructor: deep copy ทั้ง buffer -- แพงมากถ้า size_ ใหญ่
    BigBuffer(const BigBuffer& other) : size_(other.size_), data_(new int[other.size_]) {
        std::cout << "  [COPY]  ก๊อปปี้ข้อมูลทั้งหมด " << size_ << " int (แพง!)\n";
        for (std::size_t i = 0; i < size_; ++i) data_[i] = other.data_[i];
    }

    // Move constructor: แค่ "ขโมย" pointer มาใช้ ไม่ก๊อปปี้ข้อมูลเลยสักตัว
    BigBuffer(BigBuffer&& other) noexcept
        : size_(std::exchange(other.size_, 0)), data_(std::exchange(other.data_, nullptr)) {
        std::cout << "  [MOVE]  ย้าย pointer เฉยๆ ไม่แตะข้อมูลเลย (ถูกมาก)\n";
    }

    std::size_t size() const { return size_; }

private:
    std::size_t size_;
    int* data_;
};

int main() {
    std::cout << "-- push_back เข้า vector ที่จองพื้นที่ไม่พอ ต้อง reallocate --\n";
    std::vector<BigBuffer> buffers;
    buffers.reserve(1); // จงใจให้พื้นที่ไม่พอ เพื่อบังคับให้เกิด reallocation ตอน push_back ตัวที่ 2

    buffers.emplace_back(1'000'000); // สร้างตัวแรกตรงๆ ใน vector ไม่มี copy/move
    std::cout << "-- กำลัง push_back ตัวที่สอง (จะ trigger reallocation) --\n";
    buffers.emplace_back(2'000'000);
    // เพราะ move constructor ของ BigBuffer เป็น noexcept, std::vector รู้ว่า "ย้ายได้อย่างปลอดภัย"
    // จึงเลือก MOVE ตัวเก่าไปที่หน่วยความจำใหม่แทนที่จะ COPY (ทบทวนกับดักเรื่อง noexcept ด้านล่าง)

    std::cout << "จำนวน buffer ทั้งหมด = " << buffers.size() << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 02_why_move.cpp -o 02_move
./02_move
```

```
-- push_back เข้า vector ที่จองพื้นที่ไม่พอ ต้อง reallocate --
  [สร้างใหม่] จอง 1000000 int
-- กำลัง push_back ตัวที่สอง (จะ trigger reallocation) --
  [สร้างใหม่] จอง 2000000 int
  [MOVE]  ย้าย pointer เฉยๆ ไม่แตะข้อมูลเลย (ถูกมาก)
จำนวน buffer ทั้งหมด = 2
```

สังเกตว่าตอน `emplace_back` ตัวที่สองทำให้ `vector` ต้อง reallocate หน่วยความจำใหม่ (เพราะ
`reserve(1)` จองที่ไว้แค่ 1 ช่อง) `BigBuffer` ตัวแรก (ที่มีข้อมูลถึง 1,000,000 int) ต้องถูกย้าย
จากหน่วยความจำเก่าไปยังหน่วยความจำใหม่ — ถ้าไม่มี move constructor ตรงนี้จะต้อง **copy ข้อมูล
ทั้ง 1,000,000 ตัวเลข** แต่เพราะเรามี move constructor ที่ทำแค่ "สลับ pointer" จึงเห็นข้อความ
`[MOVE]` แทน โดยใช้เวลาแทบจะเท่ากับ 0 ไม่ว่า buffer จะมีขนาดใหญ่แค่ไหนก็ตาม

### ทำไม noexcept ถึงสำคัญขนาดนี้กับ move constructor

จุดที่หลายคนพลาดบ่อยที่สุดคือการลืมใส่ `noexcept` ให้ move constructor ลองดูว่าเกิดอะไรขึ้นถ้า
ลืมใส่:

```cpp
// 02b_noexcept_matters.cpp - ถ้า move constructor ไม่ใส่ noexcept, vector จะ "ไม่กล้า" ใช้ move ตอน reallocate
#include <iostream>
#include <type_traits>
#include <utility>
#include <vector>

class RiskyBuffer {
public:
    explicit RiskyBuffer(std::size_t n) : size_(n), data_(new int[n]) {}
    ~RiskyBuffer() { delete[] data_; }

    RiskyBuffer(const RiskyBuffer& other) : size_(other.size_), data_(new int[other.size_]) {
        std::cout << "  [COPY]  (เพราะ move ไม่ใช่ noexcept, vector เลือก copy เพื่อความปลอดภัย)\n";
        for (std::size_t i = 0; i < size_; ++i) data_[i] = other.data_[i];
    }

    // ตั้งใจ "ไม่ใส่ noexcept" เพื่อสาธิตกับดัก
    RiskyBuffer(RiskyBuffer&& other) : size_(std::exchange(other.size_, 0)),
                                        data_(std::exchange(other.data_, nullptr)) {
        std::cout << "  [MOVE]  (แต่ vector จะไม่เลือกใช้ทางนี้ตอน reallocate!)\n";
    }

    std::size_t size() const { return size_; }

private:
    std::size_t size_;
    int* data_;
};

int main() {
    std::cout << std::boolalpha;
    std::cout << "is_nothrow_move_constructible<RiskyBuffer> = "
              << std::is_nothrow_move_constructible_v<RiskyBuffer> << '\n';

    std::vector<RiskyBuffer> buffers;
    buffers.reserve(1);
    buffers.emplace_back(10);
    std::cout << "-- push_back ตัวที่สอง (reallocate) --\n";
    buffers.emplace_back(20); // จะเห็น [COPY] แทน [MOVE] แม้จะมี move constructor ให้ใช้ก็ตาม

    std::cout << "จำนวน buffer = " << buffers.size() << '\n';
    return 0;
}
```

```
is_nothrow_move_constructible<RiskyBuffer> = false
-- push_back ตัวที่สอง (reallocate) --
  [COPY]  (เพราะ move ไม่ใช่ noexcept, vector เลือก copy เพื่อความปลอดภัย)
จำนวน buffer = 2
```

เหตุผลที่ `std::vector` ทำแบบนี้คือเรื่อง **exception safety** (ทบทวน Part 68.7): ถ้า move
constructor throw exception กลางทางระหว่างการ reallocate `vector` จะตกอยู่ในสถานะที่กู้คืน
ไม่ได้เลย (บาง element ถูกย้ายไปที่ใหม่แล้ว บาง element ยังอยู่ที่เก่า) แต่ถ้าใช้ copy แทน ถึง
แม้ copy จะ throw กลางทาง ต้นฉบับก็ยังอยู่ครบถ้วนสมบูรณ์ ทำให้ `vector` กู้คืนสถานะเดิมได้ (strong
guarantee) `std::vector` จึงใช้กฎอนุรักษ์นิยมนี้เสมอ: **จะเลือกใช้ move ตอน reallocate ก็ต่อเมื่อ
move constructor รับประกันว่าจะไม่ throw เท่านั้น (`noexcept`)** ถ้าไม่มั่นใจ มันจะถอยไปใช้ copy
เสมอ แม้จะรู้ว่ามี move constructor ให้ใช้ก็ตาม

---

## 70.3 Rvalue Reference (&&) (Step 555)

C++11 เพิ่มรูปแบบ reference ใหม่ที่เขียนด้วยเครื่องหมาย `&&` เรียกว่า **rvalue reference**
ต่างจาก `&` (lvalue reference) ที่เคยเรียนมาตั้งแต่ Part 43 ตรงที่มันสามารถ **ผูกกับ rvalue
เท่านั้น** และเปิดโอกาสให้เขียน overload ของฟังก์ชันที่แยกแยะว่าอาร์กิวเมนต์ที่ส่งเข้ามาเป็น
lvalue หรือ rvalue ได้

```cpp
// 03_rvalue_ref.cpp
#include <iostream>
#include <string>

void process(const std::string& s) {   // รับได้ทั้ง lvalue และ rvalue แต่ "ไม่รู้" ว่าอันไหนเป็นอันไหน
    std::cout << "  [lvalue ref (const&)] รับค่า: " << s << '\n';
}

void process(std::string&& s) {        // overload นี้จะถูกเลือกเฉพาะเมื่ออาร์กิวเมนต์เป็น rvalue เท่านั้น
    std::cout << "  [rvalue ref (&&)]   รับค่าที่ปลอดภัยจะ \"ขโมย\": " << s << '\n';
}

int main() {
    std::string name = "Somchai";

    process(name);                       // name เป็น lvalue (มีชื่อ) -> เรียก overload (const&)
    process(std::string("temporary"));   // ค่าชั่วคราวเป็น rvalue -> เรียก overload (&&)
    process(name + "!");                 // ผลลัพธ์ของ operator+ เป็น rvalue ชั่วคราว -> เรียก overload (&&)

    // ตัวแปร rvalue reference เอง เมื่อมีชื่อแล้ว กลับกลายเป็น lvalue!
    std::string&& rref = std::string("hello");
    process(rref);          // rref มีชื่อแล้ว จึงเป็น lvalue -> เรียก overload (const&) ไม่ใช่ (&&)

    return 0;
}
```

```
  [lvalue ref (const&)] รับค่า: Somchai
  [rvalue ref (&&)]   รับค่าที่ปลอดภัยจะ "ขโมย": temporary
  [rvalue ref (&&)]   รับค่าที่ปลอดภัยจะ "ขโมย": Somchai!
  [lvalue ref (const&)] รับค่า: hello
```

จุดที่มักทำให้สับสนที่สุดในหัวข้อนี้คือบรรทัดสุดท้าย: `rref` **ถูกประกาศด้วยชนิด**
`std::string&&` ก็จริง แต่พอมันมีชื่อแล้ว (`rref` คือชื่อของมัน) **ตัวมันเองกลายเป็น lvalue**
เมื่อถูกใช้งานในนิพจน์อื่นต่อไป — นี่คือกฎที่สำคัญมาก:

> **กฎจำง่าย**: `T&&` เป็นแค่ **ชนิดของ reference** (rvalue reference type) แต่ **ตัวแปรที่มี
> ชื่อทุกตัวเป็น lvalue เสมอ** ไม่ว่าจะประกาศด้วยชนิด reference แบบใดก็ตาม การจะ "ส่งต่อ" มันไป
> เป็น rvalue อีกครั้งต้องใช้ `std::move` อย่างชัดเจนเท่านั้น (จะอธิบายในหัวข้อถัดไป)

---

## 70.4 std::move: แค่ Cast ไม่ได้ "ย้าย" อะไรด้วยตัวมันเอง (Step 556)

นี่คือความเข้าใจผิดที่พบบ่อยที่สุดในหมู่ผู้เริ่มต้นเรียน C++ สมัยใหม่: **`std::move` ไม่ได้
ย้ายข้อมูลอะไรเลยแม้แต่นิดเดียว** ชื่อของมันทำให้เข้าใจผิดได้ง่ายมาก แต่ในความเป็นจริง
`std::move(x)` **แค่ทำหน้าที่ static_cast ให้ `x` (ซึ่งเป็น lvalue) มองว่าเป็น rvalue reference**
เท่านั้นเอง เปรียบเสมือนการติดป้าย "ของนี้ทิ้งได้" บนกล่องใบหนึ่ง — การติดป้ายไม่ได้ทำให้ของใน
กล่องหายไปไหน มันแค่บอกให้คนที่มาเจอกล่องนี้ทีหลังรู้ว่า **สามารถหยิบของข้างในไปใช้ได้เลยโดยไม่
ต้องขออนุญาตหรือคืนของ**

```cpp
// 04_std_move.cpp - std::move คือแค่การ cast ไม่ได้ "ย้าย" อะไรด้วยตัวมันเอง
#include <iostream>
#include <string>
#include <type_traits>
#include <utility>

// ประมาณเทียบเท่าของ std::move ที่ Standard Library ให้มา (แค่ static_cast เท่านั้น!)
template <typename T>
constexpr std::remove_reference_t<T>&& myMove(T&& value) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(value);
}

int main() {
    std::string a = "hello world, this is a somewhat long string";

    // std::move(a) แค่ "เปลี่ยนมุมมอง" ของ a ให้ compiler มองเป็น rvalue reference
    // มันไม่ได้ทำอะไรกับ a เลยแม้แต่นิดเดียว ณ บรรทัดนี้!
    std::string&& viewAsRvalue = std::move(a);
    std::cout << "หลัง std::move (ยังไม่มีใครใช้จริง) a ยังคงเป็น: " << a << '\n';
    (void)viewAsRvalue;

    // การ "ย้าย" ที่แท้จริงเกิดขึ้นก็ต่อเมื่อมีบางอย่าง (เช่น move constructor) "รับ" ค่านั้นไปใช้
    std::string b = std::move(a);   // ตอนนี้ต่างหากที่ std::string::string(string&&) ทำการขโมยข้อมูลจริง

    std::cout << "b = " << b << '\n';
    std::cout << "a หลังถูก move ไปแล้วจริง: \"" << a << "\" (สถานะไม่แน่นอน แต่ valid เสมอ)\n";

    // ใช้ myMove ที่เขียนเลียนแบบเองเพื่อยืนยันว่ามันคือแค่ cast จริงๆ
    std::string c = "อีกก้อนหนึ่ง";
    std::string d = myMove(c);
    std::cout << "d = " << d << ", c หลัง myMove = \"" << c << "\"\n";

    return 0;
}
```

```
หลัง std::move (ยังไม่มีใครใช้จริง) a ยังคงเป็น: hello world, this is a somewhat long string
b = hello world, this is a somewhat long string
a หลังถูก move ไปแล้วจริง: "" (สถานะไม่แน่นอน แต่ valid เสมอ)
d = อีกก้อนหนึ่ง, c หลัง myMove = ""
```

สังเกตผลลัพธ์บรรทัดที่ 1: หลังเรียก `std::move(a)` แต่ยังไม่มีใครนำค่านั้นไป "รับ" จริงๆ (แค่ผูก
กับ `viewAsRvalue` ซึ่งเป็นแค่ reference ที่ยังไม่ได้ trigger การสร้าง object ใหม่) `a` ยัง
เหมือนเดิมทุกประการ การเปลี่ยนแปลงจริงเกิดขึ้นที่บรรทัด `std::string b = std::move(a);`
ต่างหาก เพราะบรรทัดนี้เรียก **move constructor ของ `std::string`** ซึ่งเป็นฟังก์ชันที่ทำการ
"ขโมย" buffer ภายในของ `a` มาให้ `b` จริงๆ (และเซ็ตสถานะภายในของ `a` ให้ว่างเปล่า)

> **สรุปแก่นความคิดที่สำคัญที่สุดของหัวข้อนี้**: `std::move` เป็นแค่ **สัญญาณบอก compiler** ว่า
> "อนุญาตให้เลือก overload แบบ rvalue reference (`&&`) ได้" การย้ายทรัพยากรที่แท้จริงเกิดขึ้นที่
> **โค้ดภายใน move constructor/move assignment operator** ที่ถูกเลือกให้ทำงานเพราะสัญญาณนั้น
> ต่างหาก — ถ้า type นั้นไม่มี move constructor เลย `std::move` จะไม่มีผลอะไรนอกจากทำให้
> compiler เลือก **copy constructor** แทน (จะเห็นตัวอย่างชัดเจนในหัวข้อ 70.6)

### สถานะของ Object หลังถูก Move: "Moved-From State"

มาตรฐาน C++ รับประกันแค่ว่า object ที่ถูก move ไปแล้ว (เรียกว่าอยู่ใน **moved-from state**)
จะยังอยู่ใน **สถานะที่ valid** (เรียก destructor ได้ปลอดภัย, กำหนดค่าใหม่ให้มันได้) แต่
**ไม่รับประกันว่าค่าข้างในจะเป็นอะไร** — สำหรับ `std::string`/`std::vector` มาตรฐานในทางปฏิบัติ
(แม้ไม่ได้การันตีอย่างเป็นทางการ 100% ในทุก implementation) มักจะทำให้มันกลายเป็นค่าว่าง
(`empty()` เป็น `true`) แต่ **ไม่ควรเขียนโค้ดที่พึ่งพาพฤติกรรมนี้** ควรถือว่า object หลัง move
"อยู่ในสถานะที่ไม่แน่นอน" เสมอ จนกว่าจะกำหนดค่าใหม่ให้มัน

---

## 70.5 เขียน Move Constructor และ Move Assignment Operator เอง (Step 557)

ตอนนี้เราเข้าใจทฤษฎีครบแล้ว มาถึงเวลาเขียน class ที่ถือ raw pointer เองแบบเต็มรูปแบบ พร้อม
ทั้ง move constructor และ move assignment operator (ทบทวนสไตล์ class จาก **Part 51 —
Operator Overloading**)

```cpp
// 05_move_ctor_assign.cpp - เขียน move constructor / move assignment เองสำหรับ class ที่ถือ raw pointer
// (ทบทวนสไตล์ class จาก Part 51 - Operator Overloading / IntArray)
#include <cstring>
#include <iostream>
#include <utility>

class Message {
public:
    explicit Message(const char* text) : len_(std::strlen(text)), data_(new char[len_ + 1]) {
        std::memcpy(data_, text, len_ + 1);
        std::cout << "  [constructor] สร้าง Message(\"" << data_ << "\")\n";
    }

    ~Message() {
        if (data_) std::cout << "  [destructor]  ทำลาย Message(\"" << data_ << "\")\n";
        delete[] data_;
    }

    // ---- Copy constructor: deep copy ----
    Message(const Message& other) : len_(other.len_), data_(new char[other.len_ + 1]) {
        std::memcpy(data_, other.data_, len_ + 1);
        std::cout << "  [copy ctor]   deep copy จาก \"" << other.data_ << "\"\n";
    }

    // ---- Copy assignment: deep copy พร้อมป้องกัน self-assignment ----
    Message& operator=(const Message& other) {
        std::cout << "  [copy assign] deep copy จาก \"" << other.data_ << "\"\n";
        if (this != &other) {
            char* newData = new char[other.len_ + 1];
            std::memcpy(newData, other.data_, other.len_ + 1);
            delete[] data_;
            data_ = newData;
            len_ = other.len_;
        }
        return *this;
    }

    // ---- Move constructor: ขโมย pointer แล้วเคลียร์ต้นทาง ต้องเป็น noexcept เสมอ ----
    Message(Message&& other) noexcept
        : len_(std::exchange(other.len_, 0)), data_(std::exchange(other.data_, nullptr)) {
        std::cout << "  [move ctor]   ขโมย buffer มาโดยไม่ copy ข้อมูลเลย\n";
    }

    // ---- Move assignment: คืนของเดิมก่อน แล้วขโมยของใหม่ ต้องเป็น noexcept เสมอ ----
    Message& operator=(Message&& other) noexcept {
        std::cout << "  [move assign] ขโมย buffer มาแทนที่ของเดิม\n";
        if (this != &other) {
            delete[] data_;
            len_ = std::exchange(other.len_, 0);
            data_ = std::exchange(other.data_, nullptr);
        }
        return *this;
    }

    const char* text() const { return data_ ? data_ : "(moved-from / ว่างเปล่า)"; }

private:
    std::size_t len_;
    char* data_;
};

int main() {
    Message a("สวัสดี");
    Message b = a;                 // copy constructor
    Message c = std::move(a);      // move constructor -- a ถูกขโมยข้อมูลไปแล้ว

    std::cout << "a.text() หลังถูก move = \"" << a.text() << "\"\n";
    std::cout << "c.text() = \"" << c.text() << "\"\n";

    Message d("ตัวชั่วคราว");
    d = b;                          // copy assignment
    Message e("จะถูกแทนที่");
    e = std::move(c);                // move assignment -- c ถูกขโมยข้อมูลไปแล้ว

    std::cout << "e.text() = \"" << e.text() << "\", c.text() หลังถูก move = \"" << c.text() << "\"\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 05_move_ctor_assign.cpp -o 05_move
./05_move
```

```
  [constructor] สร้าง Message("สวัสดี")
  [copy ctor]   deep copy จาก "สวัสดี"
  [move ctor]   ขโมย buffer มาโดยไม่ copy ข้อมูลเลย
a.text() หลังถูก move = "(moved-from / ว่างเปล่า)"
c.text() = "สวัสดี"
  [constructor] สร้าง Message("ตัวชั่วคราว")
  [copy assign] deep copy จาก "สวัสดี"
  [constructor] สร้าง Message("จะถูกแทนที่")
  [move assign] ขโมย buffer มาแทนที่ของเดิม
e.text() = "สวัสดี", c.text() หลังถูก move = "(moved-from / ว่างเปล่า)"
  [destructor]  ทำลาย Message("สวัสดี")
  [destructor]  ทำลาย Message("สวัสดี")
  [destructor]  ทำลาย Message("สวัสดี")
```

สังเกตรูปแบบร่วมของทั้ง move constructor และ move assignment operator ทั้งสองใช้เทคนิคเดียวกัน
คือ **`std::exchange`** (จาก `<utility>`, เคยเห็นมาแล้วใน Part 68.4) ซึ่งทำสองอย่างพร้อมกันใน
บรรทัดเดียว: อ่านค่าเดิมออกมา และตั้งค่าใหม่ให้ต้นทาง (`other`) เป็นสถานะว่างเปล่า
(`nullptr`/`0`) — นี่คือสำนวนมาตรฐานที่ใช้เขียน move operation แทบทุกที่ในโค้ด C++ สมัยใหม่
เพราะป้องกันไม่ให้เกิด **double free** ตอนที่ `other` หมด scope แล้ว destructor ของมันพยายาม
`delete[]` ข้อมูลที่ถูกขโมยไปแล้ว

### จุดสำคัญที่ต้องระวังในการเขียน Move Operation

1. **Move constructor และ move assignment ต้องเป็น `noexcept` เสมอ** (ตามที่อธิบายไปแล้วใน
   70.2) — ไม่มีข้อยกเว้น เว้นแต่มีเหตุผลที่หนักแน่นจริงๆ ว่าทำไมมันถึง throw ได้ (ซึ่งพบได้ยาก
   มากในทางปฏิบัติ)
2. **ต้องเช็ค self-assignment ใน move assignment** (`if (this != &other)`) แม้ในทางทฤษฎี
   `x = std::move(x)` จะเป็นโค้ดที่แปลกและไม่ควรเกิดขึ้น แต่ในทางปฏิบัติ (เช่นผ่าน alias หรือ
   reference สองตัวที่ชี้ไปที่เดียวกัน) เคสนี้เกิดขึ้นได้จริง ถ้าไม่เช็คไว้ โค้ดข้างบนจะ
   `delete[] data_` ทิ้งไปก่อนที่จะพยายามอ่าน `other.data_` (ซึ่งเป็นตัวเดียวกัน) ทำให้เกิด
   use-after-free ทันที
3. **ต้องเซ็ตสถานะของ `other` ให้เป็น "ว่างเปล่าที่ปลอดภัย" เสมอหลัง move** (`nullptr`, `0`)
   เพื่อให้ destructor ของ `other` ทำงานได้อย่างปลอดภัยโดยไม่ทำอะไรเลย (การ `delete[] nullptr`
   ปลอดภัยเสมอตามมาตรฐานภาษา ไม่ทำให้เกิด undefined behavior)

---

## 70.6 Rule of Five: ขยายจาก Rule of Three (Step 558)

ทบทวนจาก **Part 46, 51, และ 68.6**: **Rule of Three** (กฎที่ใช้มาตั้งแต่ยุค C++98) กล่าวว่า
ถ้า class ต้องเขียน destructor เอง (เพราะถือ raw resource) มันมักจะต้องเขียน copy constructor
และ copy assignment operator เองด้วยเสมอ เพราะ compiler-generated copy จะแค่ copy pointer
ตรงๆ (shallow copy) นำไปสู่ double free

เมื่อ C++11 เพิ่ม move semantics เข้ามา กฎนี้ขยายกลายเป็น **Rule of Five**: class ที่ถือ raw
resource เอง ควรเขียนให้ครบทั้ง **5 ฟังก์ชันพิเศษ**:

| # | ฟังก์ชัน | หน้าที่ |
|---|---|---|
| 1 | Destructor `~T()` | คืนทรัพยากรเมื่อ object หมดอายุ |
| 2 | Copy constructor `T(const T&)` | สร้าง object ใหม่โดย deep copy จากต้นฉบับ |
| 3 | Copy assignment `T& operator=(const T&)` | แทนที่เนื้อหาของ object เดิมด้วย deep copy จากอีกตัว |
| 4 | Move constructor `T(T&&) noexcept` | สร้าง object ใหม่โดย "ขโมย" ทรัพยากรจากต้นฉบับที่กำลังจะถูกทิ้ง |
| 5 | Move assignment `T& operator=(T&&) noexcept` | แทนที่เนื้อหาของ object เดิมโดยขโมยทรัพยากรจากอีกตัว |

### ทำไม "แค่ Rule of Three" ถึงไม่พออีกต่อไปในโลกที่มี Move Semantics

จุดสำคัญที่ต้องเข้าใจให้ลึกคือ **ถ้า class เขียนแค่ destructor + copy (Rule of Three) โดยไม่
เขียน move เลย compiler จะ "ไม่สร้าง" move constructor/assignment ให้อัตโนมัติ** (กฎของภาษา:
การมี user-declared copy constructor, copy assignment, หรือ destructor อย่างใดอย่างหนึ่ง จะปิด
การสร้าง move operation แบบอัตโนมัติทันที) ผลคือเมื่อมีใครเรียก `std::move` กับ object ของ
class นั้น **compiler จะถอยไปเรียก copy constructor แทนอย่างเงียบๆ** โดยไม่มี error หรือ
warning ใดๆ เลย — โค้ดยังคอมไพล์ผ่านและทำงานถูกต้อง แต่ **เสียประสิทธิภาพไปฟรีๆ** เพราะสิ่งที่
ตั้งใจจะให้เป็น "ย้าย" (เร็ว) กลายเป็น "copy" (ช้า) โดยไม่มีใครรู้ตัว

```cpp
// 06_rule_of_five.cpp - สาธิตว่าทำไม Rule of Three แบบเดิมไม่พอในโลกที่มี move semantics
#include <iostream>
#include <type_traits>
#include <utility>

// เขียนแค่ destructor + copy (สไตล์ Rule of Three ก่อน C++11) โดยไม่เขียน move เลย
class OldStyleBuffer {
public:
    explicit OldStyleBuffer(std::size_t n) : size_(n), data_(new int[n]{}) {}
    ~OldStyleBuffer() { delete[] data_; }
    OldStyleBuffer(const OldStyleBuffer& other) : size_(other.size_), data_(new int[other.size_]) {
        std::cout << "  [OldStyleBuffer copy ctor] deep copy (ไม่มี move ให้เลือกใช้เลย)\n";
        for (std::size_t i = 0; i < size_; ++i) data_[i] = other.data_[i];
    }
    OldStyleBuffer& operator=(const OldStyleBuffer&) = default; // สมมติแบบง่าย (ไม่ได้ใช้ในตัวอย่างนี้)
    std::size_t size() const { return size_; }

private:
    std::size_t size_;
    int* data_;
};

int main() {
    std::cout << std::boolalpha;
    std::cout << "OldStyleBuffer มี move constructor ที่ compiler generate ให้หรือไม่ (is_move_constructible): "
              << std::is_move_constructible_v<OldStyleBuffer> << '\n';
    // (หมายเหตุ: is_move_constructible เป็น true ได้แม้ไม่มี move ctor เพราะ overload resolution
    //  จะ fallback ไปเรียก copy constructor แทนโดยอัตโนมัติ ซึ่งนั่นแหละคือปัญหา!)

    OldStyleBuffer a(5);
    OldStyleBuffer b = std::move(a); // เขียน std::move ไว้ชัดเจน แต่จริงๆ แล้ว "ได้ copy" ไม่ใช่ move!
                                     // เพราะไม่มี move constructor ให้เลือก compiler จึงถอยไปใช้ copy ctor แทน
    std::cout << "b.size() = " << b.size() << '\n';

    return 0;
}
```

```
OldStyleBuffer มี move constructor ที่ compiler generate ให้หรือไม่ (is_move_constructible): true
  [OldStyleBuffer copy ctor] deep copy (ไม่มี move ให้เลือกใช้เลย)
b.size() = 5
```

ผลลัพธ์นี้คือสิ่งที่น่ากลัวที่สุดของปัญหานี้: `std::is_move_constructible_v<OldStyleBuffer>`
ให้ค่า `true` (เพราะในทางเทคนิค "การเรียกด้วย rvalue แล้ว compile ผ่าน" นับว่า move-constructible
ได้ แม้จริงๆ แล้วมันจะ fallback ไปเรียก copy constructor ก็ตาม) โปรแกรมยังทำงานถูกต้อง 100%
ไม่มี crash ไม่มี error แต่ **สูญเสียประสิทธิภาพที่ควรได้จาก move ไปอย่างเงียบๆ** — นี่คือเหตุผล
ที่ Part 68.6 เคยเตือนไว้แล้วว่า **ห้ามเขียน destructor เปล่าๆ ทิ้งไว้โดยไม่จำเป็น** เพราะมันปิด
การสร้าง move operation อัตโนมัติของ compiler แม้จะไม่ได้ตั้งใจก็ตาม

### ตารางสรุป Rule of Zero / Three / Five (ฉบับสมบูรณ์)

| กฎ | ใช้เมื่อ | ตัวอย่าง |
|---|---|---|
| **Rule of Zero** | Class ไม่ถือ raw resource เอง สมาชิกทุกตัวเป็น RAII type (`string`, `vector`, smart pointer) | `Report` (Part 68.5), เกือบทุก class ระดับ business logic |
| **Rule of Three** | Class ถือ raw resource เอง (ไม่สนใจ move เลย หรือทำงานในโค้ดเบส C++98/03) | โค้ดเก่าก่อน C++11, หรือกรณีพิเศษที่ move ไม่มีความหมาย |
| **Rule of Five** | Class ถือ raw resource เอง และต้องการรองรับ move semantics เพื่อประสิทธิภาพสูงสุด | `Message`, `BigBuffer` ในบทเรียนนี้, `FileGuard`/`SocketGuard` (Part 68.2, 68.4) |

> **กฎทองของ Part นี้**: ในโค้ด C++11 ขึ้นไปทุกจุดที่ต้องเขียน destructor เอง (เพราะถือ raw
> resource) **ให้เขียน Rule of Five ให้ครบทั้ง 5 ฟังก์ชันเสมอ** ไม่ใช่แค่ Rule of Three — ถ้า
> ไม่แน่ใจว่า move ควรทำงานอย่างไร อย่างน้อยที่สุดให้เขียน `= delete` สำหรับ copy และปล่อยให้
> compiler generate move ให้ (ถ้าเป็นไปได้) แทนที่จะปล่อยให้เกิดการ fallback ไปเป็น copy อย่าง
> เงียบๆ โดยไม่รู้ตัว

---

## 70.7 std::swap และ Copy-and-Swap Idiom (Step 559)

`std::swap(a, b)` (จาก `<utility>`) เป็นฟังก์ชันพื้นฐานที่สุดที่สลับค่าของสอง object เข้าด้วยกัน
ในเวอร์ชันสมัยใหม่ (C++11 ขึ้นไป) `std::swap` ถูก implement ด้วย move semantics เอง (ย้าย `a`
ไปตัวแปรชั่วคราว ย้าย `b` มาที่ `a` แล้วย้ายตัวแปรชั่วคราวไปที่ `b`) ทำให้มันเร็วมากสำหรับ type
ที่มี move constructor ที่ดี

เทคนิคที่ใช้ `swap` เป็นแกนกลางที่ทรงพลังที่สุดอันหนึ่งใน C++ คือ **copy-and-swap idiom** ที่
เคยเกริ่นไว้ใน Part 68.7 เรื่อง strong exception guarantee — แต่ในหัวข้อนี้เราจะโฟกัสที่ประโยชน์
อีกด้านของมัน: **การเขียน `operator=` แค่ตัวเดียวที่รองรับได้ทั้ง copy assignment และ move
assignment พร้อมกัน**

```cpp
// 07_copy_and_swap.cpp - std::swap และ copy-and-swap idiom
#include <algorithm>
#include <iostream>
#include <utility>

class IntArray {
public:
    explicit IntArray(std::size_t size) : size_(size), data_(new int[size]{}) {}

    ~IntArray() { delete[] data_; }

    IntArray(const IntArray& other) : size_(other.size_), data_(new int[other.size_]) {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    IntArray(IntArray&& other) noexcept : size_(0), data_(nullptr) {
        swap(*this, other);
    }

    // copy-and-swap idiom: ใช้ operator= ตัวเดียวรองรับได้ทั้ง copy และ move!
    // รับพารามิเตอร์ "by value" (จะเรียก copy ctor หรือ move ctor โดยอัตโนมัติแล้วแต่ argument)
    // จากนั้น swap กับสำเนาที่ได้มา -- ให้ strong exception guarantee ฟรี (Part 68.7)
    IntArray& operator=(IntArray other) {
        swap(*this, other);
        return *this;
        // other (สำเนาที่รับเข้ามา) จะถูกทำลายตอนจบฟังก์ชัน พร้อมพา data_ เก่าของ *this ไปทิ้งด้วย
    }

    friend void swap(IntArray& a, IntArray& b) noexcept {
        std::swap(a.size_, b.size_);
        std::swap(a.data_, b.data_);
    }

    std::size_t size() const { return size_; }
    int& operator[](std::size_t i) { return data_[i]; }

private:
    std::size_t size_;
    int* data_;
};

int main() {
    IntArray a(3);
    a[0] = 1; a[1] = 2; a[2] = 3;

    IntArray b(1);
    b = a;                     // เรียก operator=(IntArray) -> copy ctor สร้างสำเนา แล้ว swap
    std::cout << "b.size() = " << b.size() << ", b[1] = " << b[1] << '\n';

    IntArray c(1);
    c = std::move(a);          // เรียก operator=(IntArray) เหมือนกัน แต่รอบนี้ argument เป็น move ctor สร้างสำเนา
    std::cout << "c.size() = " << c.size() << ", c[1] = " << c[1] << '\n';
    std::cout << "a.size() หลัง move = " << a.size() << " (ถูกขโมยไปหมดแล้ว)\n";

    IntArray d(5);
    IntArray e(2);
    swap(d, e);                 // เรียก swap แบบ noexcept ตรงๆ ไม่ผ่าน operator= เลย
    std::cout << "หลัง swap(d, e): d.size() = " << d.size() << ", e.size() = " << e.size() << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 07_copy_and_swap.cpp -o 07_cs
./07_cs
```

```
b.size() = 3, b[1] = 2
c.size() = 3, c[1] = 2
a.size() หลัง move = 0 (ถูกขโมยไปหมดแล้ว)
หลัง swap(d, e): d.size() = 2, e.size() = 5
```

กลไกที่ทำให้ `operator=(IntArray other)` ตัวเดียวรองรับได้ทั้งสองแบบคือการ **รับพารามิเตอร์ by
value** — เมื่อเรียก `b = a` (โดยที่ `a` เป็น lvalue) compiler จะเลือกเรียก **copy constructor**
เพื่อสร้างพารามิเตอร์ `other` ขึ้นมา แต่เมื่อเรียก `c = std::move(a)` (โดยที่อาร์กิวเมนต์เป็น
rvalue reference) compiler จะเลือกเรียก **move constructor** แทนโดยอัตโนมัติ — ไม่ว่าจะทางไหน
เมื่อได้ `other` มาแล้ว เราแค่ `swap` มันกับ `*this` ตัวเดียวจบ จึงไม่ต้องเขียน copy assignment
และ move assignment แยกกันสองฟังก์ชันอีกต่อไป

> **ข้อควรระวัง**: copy-and-swap idiom แลกความกระชับของโค้ดกับประสิทธิภาพในบางกรณี — ถ้าเดิม
> class มี copy assignment ที่ optimize ให้ **reuse หน่วยความจำเดิม** ได้ (เช่นถ้าขนาดใหม่
> เล็กกว่าหรือเท่าเดิม ก็ copy ทับได้เลยไม่ต้อง allocate ใหม่) การเปลี่ยนมาใช้ copy-and-swap จะ
> เสีย optimization นั้นไป เพราะมันสร้างสำเนาใหม่ (allocate ใหม่) เสมอทุกครั้ง ควรใช้เทคนิคนี้
> เมื่อความง่ายในการรักษา exception safety สำคัญกว่าการ optimize กรณีพิเศษเหล่านั้น

---

## 70.8 Perfect Forwarding เบื้องต้น: std::forward และ Forwarding Reference (Step 560)

ปัญหาสุดท้ายที่ยังไม่ได้แก้ในหัวข้อนี้คือ: ถ้าเราเขียนฟังก์ชัน **wrapper** ที่รับพารามิเตอร์แล้ว
"ส่งต่อ" ไปให้ฟังก์ชันอื่นอีกที (เช่น logging wrapper, factory function) จะทำอย่างไรให้ข้อมูล
เรื่อง "เป็น lvalue หรือ rvalue" ของอาร์กิวเมนต์เดิม **ไม่หายไปกลางทาง**?

```cpp
// 08_perfect_forwarding.cpp
#include <iostream>
#include <string>
#include <utility>

void process(const std::string& s) {
    std::cout << "  [lvalue ref] " << s << '\n';
}
void process(std::string&& s) {
    std::cout << "  [rvalue ref] " << s << '\n';
}

// (1) ฟังก์ชันที่ "ไม่" forward อย่างถูกต้อง: parameter ชื่อ arg เป็น lvalue เสมอ (มีชื่อ = lvalue)
// ไม่ว่าผู้เรียกจะส่ง lvalue หรือ rvalue เข้ามา process(arg) จะเรียก overload (const&) เสมอ
template <typename T>
void badWrapper(T&& arg) {
    std::cout << "badWrapper ส่งต่อแบบ lvalue เสมอ:\n";
    process(arg);
}

// (2) ฟังก์ชันที่ forward อย่างถูกต้องด้วย std::forward: รักษาความเป็น lvalue/rvalue เดิมไว้
// T&& ตรงนี้เรียกว่า "forwarding reference" (หรือชื่อเดิม "universal reference")
// เพราะ T ถูกอนุมานจาก template จึงมีความหมายต่างจาก rvalue reference ธรรมดา (เช่น string&&)
template <typename T>
void goodWrapper(T&& arg) {
    std::cout << "goodWrapper ส่งต่อรักษาความเป็น lvalue/rvalue เดิม:\n";
    process(std::forward<T>(arg));
}

int main() {
    std::string name = "Somchai";

    badWrapper(name);                 // ส่ง lvalue เข้าไป
    badWrapper(std::string("temp"));  // ส่ง rvalue เข้าไป แต่ badWrapper ก็ยังเรียก (const&) เหมือนเดิม!

    goodWrapper(name);                 // ส่ง lvalue -> ควรเรียก (const&)
    goodWrapper(std::string("temp"));  // ส่ง rvalue -> ควรเรียก (&&) ได้จริง

    return 0;
}
```

```
badWrapper ส่งต่อแบบ lvalue เสมอ:
  [lvalue ref] Somchai
badWrapper ส่งต่อแบบ lvalue เสมอ:
  [lvalue ref] temp
goodWrapper ส่งต่อรักษาความเป็น lvalue/rvalue เดิม:
  [lvalue ref] Somchai
goodWrapper ส่งต่อรักษาความเป็น lvalue/rvalue เดิม:
  [rvalue ref] temp
```

### ทำไม T&& ใน Template ถึงไม่ใช่ Rvalue Reference ธรรมดา

จุดที่ต้องแยกให้ออกให้ชัดคือ **`T&&` ที่ปรากฏในบริบทของ template ที่ `T` เป็น template
parameter ที่ต้องอนุมาน** มีความหมายพิเศษเรียกว่า **forwarding reference** (ชื่อเดิมที่ยังพบ
เห็นในตำราเก่าคือ "universal reference" ซึ่งเป็นศัพท์ที่ Scott Meyers บัญญัติขึ้นก่อนที่มาตรฐาน
จะใช้ชื่อทางการว่า forwarding reference) มันมีกฎพิเศษที่เรียกว่า **reference collapsing**:

| อาร์กิวเมนต์ที่ส่งเข้ามา | `T` ถูกอนุมานเป็น | `T&&` กลายเป็น |
|---|---|---|
| lvalue (เช่น `name`) | `std::string&` | `std::string& && → std::string&` (collapse เป็น lvalue ref) |
| rvalue (เช่น `std::string("temp")`) | `std::string` | `std::string&&` (คงเป็น rvalue ref) |

เพราะกลไกนี้ `T&&` ใน context นี้จึง "รับได้ทั้งสองแบบ" และรู้ว่าแบบไหนเป็นแบบไหนอยู่ภายใน (ผ่าน
`T` ที่ถูกอนุมาน) — แต่ปัญหาคือ **ตัวแปร `arg` เองมีชื่อแล้ว จึงกลายเป็น lvalue เสมอ** เหมือนที่
เจอในหัวข้อ 70.3 ทำให้ `badWrapper` ที่เรียก `process(arg)` ตรงๆ จะเรียก overload `(const&)`
เสมอ ไม่ว่าผู้เรียกจะตั้งใจส่ง rvalue มาแค่ไหนก็ตาม

`std::forward<T>(arg)` แก้ปัญหานี้โดย **"คืนความเป็น lvalue/rvalue เดิม" กลับไปให้ `arg`** โดย
อาศัยข้อมูลที่ `T` บอกไว้ (ถ้า `T` เป็น `std::string&` แปลว่าอาร์กิวเมนต์เดิมเป็น lvalue,
`std::forward` จะ cast ให้เป็น lvalue reference เหมือนเดิม แต่ถ้า `T` เป็น `std::string` เฉยๆ
(ไม่มี `&`) แปลว่าอาร์กิวเมนต์เดิมเป็น rvalue, `std::forward` จะ cast ให้เป็น rvalue reference
เหมือน `std::move`) — นี่คือที่มาของคำว่า **perfect forwarding**: ส่งต่ออาร์กิวเมนต์ไปยัง
ฟังก์ชันปลายทางโดย "รักษาทุกคุณสมบัติ" ของมันไว้อย่างสมบูรณ์แบบ ไม่มีข้อมูลสูญหายระหว่างทาง

### ตัวอย่างการใช้งานจริง: emplace_back และ make_unique

รูปแบบการใช้ perfect forwarding ที่พบบ่อยที่สุดในโค้ดจริงคือฟังก์ชันแบบ **variadic template**
ที่ส่งต่ออาร์กิวเมนต์จำนวนเท่าไหร่ก็ได้ไปสร้าง object โดยตรง ซึ่งเป็นกลไกเบื้องหลังของ
`std::vector::emplace_back` (Part 59) และ `std::make_unique` (Part 67) ทั้งคู่:

```cpp
// 09_forwarding_variadic.cpp - ตัวอย่างการใช้ perfect forwarding จริงแบบ variadic (คล้าย make_unique/emplace_back)
#include <iostream>
#include <memory>
#include <string>
#include <utility>

class Person {
public:
    Person(std::string name, int age) : name_(std::move(name)), age_(age) {
        std::cout << "  [Person ctor] สร้าง " << name_ << " อายุ " << age_ << '\n';
    }
    void print() const { std::cout << name_ << " (" << age_ << ")\n"; }

private:
    std::string name_;
    int age_;
};

// wrapper ที่ forward argument จำนวนเท่าไหร่ก็ได้ ไปยัง constructor ของ T โดยตรง
// โดยไม่ copy อะไรระหว่างทางเลยแม้แต่ครั้งเดียว -- นี่คือกลไกเบื้องหลัง std::make_unique จริงๆ
template <typename T, typename... Args>
std::unique_ptr<T> myMakeUnique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

int main() {
    auto p = myMakeUnique<Person>(std::string("Somsri"), 30);
    p->print();

    auto q = std::make_unique<Person>("Suda", 25); // ของจริงจาก <memory> ทำงานแบบเดียวกันทุกประการ
    q->print();

    return 0;
}
```

```
  [Person ctor] สร้าง Somsri อายุ 30
Somsri (30)
  [Person ctor] สร้าง Suda อายุ 25
Suda (25)
```

`Args&&...` คือ **forwarding reference แบบ variadic** (รับได้ทั้งจำนวนและชนิดของอาร์กิวเมนต์
ที่ไม่จำกัด) ร่วมกับ `std::forward<Args>(args)...` (fold expression ที่ forward แต่ละตัวแยกกัน)
ทำให้ `myMakeUnique<Person>(std::string("Somsri"), 30)` สร้าง `Person` ขึ้นมาโดยตรงโดยไม่มีการ
copy หรือ move ใดๆ เกิดขึ้นระหว่างทางเลยแม้แต่ครั้งเดียว — นี่คือเหตุผลที่ `emplace_back` และ
`make_unique`/`make_shared` มีประสิทธิภาพดีกว่าการสร้าง object แยกแล้วค่อย copy/move เข้าไปเสมอ
(ทบทวน Part 59.4 และ Part 67.3)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ object ต่อหลัง `std::move` โดยไม่ได้กำหนดค่าใหม่ให้มันก่อน** — object ที่ถูก move
   ไปแล้วอยู่ในสถานะ "moved-from" ที่ไม่แน่นอน (แม้จะยัง valid เสมอ) การเรียก method ที่คาดหวัง
   ค่าเฉพาะเจาะจง (เช่น `front()` ของ `vector` ที่อาจกลายเป็น empty แล้ว) อาจทำให้เกิด undefined
   behavior ได้ ควรกำหนดค่าใหม่ให้ object ก่อนใช้งานต่อเสมอ หรือไม่แตะมันอีกเลยจนกว่าจะหมด scope
2. **ลืมใส่ `noexcept` บน move constructor/move assignment** — ทำให้ `std::vector` (และ
   container อื่นๆ) ไม่กล้าใช้ move ตอน reallocate แล้วถอยไปใช้ copy แทนอย่างเงียบๆ (สาธิตให้
   เห็นชัดเจนในหัวข้อ 70.2) ทำลายจุดประสงค์ทั้งหมดของการเขียน move constructor ไปโดยสิ้นเชิง
3. **เขียนแค่ Rule of Three แล้วคิดว่า `std::move` จะทำงานได้เร็วขึ้นโดยอัตโนมัติ** — ถ้าไม่มี
   move constructor/assignment ที่เขียนเองหรือ compiler generate ให้ `std::move` จะแค่ทำให้
   compiler เลือก **copy constructor** แทน โดยไม่มี error หรือ warning เตือนเลย (หัวข้อ 70.6)
4. **คิดว่า `std::move` "ย้าย" ข้อมูลทันทีที่เรียก** — ที่จริงมันเป็นแค่ cast (หัวข้อ 70.4) การ
   ย้ายจริงเกิดขึ้นก็ต่อเมื่อมีบางอย่างรับค่านั้นไปสร้าง object ใหม่หรือ assign ทับ object เดิม
5. **ลืมเช็ค self-assignment ใน move assignment operator** — แม้จะดูเหมือนไม่มีทางเกิดขึ้น แต่
   ในทางปฏิบัติ code path ที่ซับซ้อน (ผ่าน reference หรือ pointer สองตัวที่ชี้ไปที่เดียวกัน)
   ทำให้เคสนี้เกิดขึ้นได้จริง และการไม่เช็คจะนำไปสู่ use-after-free ทันที
6. **ใช้ `T&` ธรรมดาแทน `T&&` (forwarding reference) ในฟังก์ชัน template ที่ตั้งใจจะ forward**
   แล้วสงสัยว่าทำไม `std::forward` ไม่ทำงานตามที่คาดหวัง — forwarding reference ต้องเขียนเป็น
   `T&&` ในบริบทที่ `T` เป็น template parameter ที่ยังไม่ถูกกำหนดตายตัว (ไม่ใช่ `std::string&&`
   ที่เป็น rvalue reference ธรรมดาซึ่งรับได้แค่ rvalue เท่านั้น)
7. **เรียก `std::forward` หรือ `std::move` มากกว่าหนึ่งครั้งกับตัวแปรเดียวกัน** — เช่น
   `foo(std::move(x)); bar(std::move(x));` — หลังบรรทัดแรก `x` อาจถูกขโมยข้อมูลไปแล้ว การเรียก
   `std::move(x)` ซ้ำในบรรทัดที่สองจะส่ง object ที่อยู่ในสถานะ moved-from ไปให้ `bar` โดยไม่ตั้งใจ

---

## แบบฝึกหัดท้ายบท

1. เขียน class `Buffer` ที่ถือ `int*` ให้ครบตาม **Rule of Five** พร้อม `noexcept` บน move
   constructor/assignment แล้วทดสอบด้วยการ `push_back`/`emplace_back` เข้า `std::vector<Buffer>`
   ที่ `reserve` พื้นที่ไว้ไม่พอ ยืนยันด้วย `std::cout` ว่าเกิด **move** ไม่ใช่ **copy** ตอน
   reallocate พร้อมใช้ `static_assert` ตรวจสอบ `std::is_nothrow_move_constructible_v<Buffer>`
2. อธิบายและแก้บั๊กในโค้ดต่อไปนี้ (ใช้ตัวแปรหลัง `std::move` โดยไม่ตรวจสอบสถานะ):
   ```cpp
   std::vector<std::string> processAndLog(std::vector<std::string> data) {
       std::vector<std::string> backup = std::move(data);
       // บั๊ก: โค้ดข้างล่างนี้ใช้ data ต่อ ทั้งที่เพิ่ง move ออกไปแล้ว!
       for (const auto& item : data) {
           std::cout << item << '\n';
       }
       return backup;
   }
   ```
3. เขียน class `Playlist` ที่มีสมาชิกเป็น `std::string owner_` และ `std::vector<std::string>
   songs_` โดยใช้ **copy-and-swap idiom** เขียน `operator=` ตัวเดียวที่รองรับได้ทั้ง copy และ
   move assignment (แม้ในทางปฏิบัติ class แบบนี้ควรใช้ Rule of Zero ก็ตาม แต่โจทย์นี้ต้องการฝึก
   ความเข้าใจกลไกของ `swap`)
4. เขียนฟังก์ชัน template `logCall` ที่รับอาร์กิวเมนต์หนึ่งตัวด้วย forwarding reference แล้วพิมพ์
   ข้อความ log ก่อน forward อาร์กิวเมนต์นั้นต่อไปยังฟังก์ชัน `target` ที่มี overload ทั้ง
   `(const std::string&)` และ `(std::string&&)` พิสูจน์ด้วย `std::cout` ว่า overload ที่ถูก
   เลือกตรงกับที่ผู้เรียก `logCall` ตั้งใจส่งมาจริง
5. วิเคราะห์นิพจน์ต่อไปนี้ทีละบรรทัดว่าเป็น lvalue หรือ rvalue พร้อมให้เหตุผล:
   `int x = 5;`, `x`, `x + 1`, `++x`, `x++`, `std::string("temp")`, `&x`
6. เขียน move assignment operator ที่ **ไม่เช็ค self-assignment** โดยตั้งใจ แล้วเขียนโค้ดทดสอบ
   ที่ทำให้เกิด self-move-assignment (`obj = std::move(obj);`) สังเกตผลลัพธ์ที่ผิดพลาด แล้วแก้ไข
   ให้ถูกต้อง

### แนวทางเฉลยข้อ 1

```cpp
// ex1_buffer_rule_of_five.cpp
#include <algorithm>
#include <iostream>
#include <type_traits>
#include <utility>
#include <vector>

class Buffer {
public:
    explicit Buffer(std::size_t size) : size_(size), data_(new int[size]{}) {
        std::cout << "  [ctor] size=" << size_ << '\n';
    }
    ~Buffer() { delete[] data_; }

    Buffer(const Buffer& other) : size_(other.size_), data_(new int[other.size_]) {
        std::cout << "  [copy ctor] size=" << size_ << '\n';
        std::copy(other.data_, other.data_ + size_, data_);
    }
    Buffer& operator=(const Buffer& other) {
        std::cout << "  [copy assign] size=" << other.size_ << '\n';
        if (this != &other) {
            int* newData = new int[other.size_];
            std::copy(other.data_, other.data_ + other.size_, newData);
            delete[] data_;
            data_ = newData;
            size_ = other.size_;
        }
        return *this;
    }

    Buffer(Buffer&& other) noexcept
        : size_(std::exchange(other.size_, 0)), data_(std::exchange(other.data_, nullptr)) {
        std::cout << "  [move ctor] size=" << size_ << '\n';
    }
    Buffer& operator=(Buffer&& other) noexcept {
        std::cout << "  [move assign] size=" << other.size_ << '\n';
        if (this != &other) {
            delete[] data_;
            size_ = std::exchange(other.size_, 0);
            data_ = std::exchange(other.data_, nullptr);
        }
        return *this;
    }

    std::size_t size() const { return size_; }

private:
    std::size_t size_;
    int* data_;
};

int main() {
    static_assert(std::is_nothrow_move_constructible_v<Buffer>,
                  "Buffer ต้อง move ได้แบบ noexcept เพื่อให้ vector reallocate ด้วย move");

    std::vector<Buffer> buffers;
    buffers.reserve(1);
    buffers.emplace_back(100);
    std::cout << "-- reallocate --\n";
    buffers.emplace_back(200); // ต้องเห็น [move ctor] ไม่ใช่ [copy ctor]

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex1_buffer_rule_of_five.cpp -o ex1
./ex1
```

```
  [ctor] size=100
-- reallocate --
  [ctor] size=200
  [move ctor] size=100
```

`static_assert` ที่ต้นฟังก์ชัน `main` เป็นเครื่องยืนยันระดับ compile-time ว่า `Buffer` ทำตาม
สัญญาที่จำเป็นสำหรับให้ `std::vector` เลือกใช้ move ตอน reallocate ได้จริง — ถ้าใครในทีมมาแก้ไข
`Buffer` ในอนาคตแล้วเผลอลบ `noexcept` ออก โปรแกรมจะ **compile ไม่ผ่านทันที** แทนที่จะปล่อยให้
บั๊กด้านประสิทธิภาพหลุดเข้าไปในโค้ด production แบบเงียบๆ

### แนวทางเฉลยข้อ 3

```cpp
// ex3_playlist_copy_swap.cpp
#include <algorithm>
#include <iostream>
#include <string>
#include <utility>
#include <vector>

class Playlist {
public:
    Playlist(std::string owner, std::vector<std::string> songs)
        : owner_(std::move(owner)), songs_(std::move(songs)) {}

    // Rule of Zero ก็พอจริงๆ (string/vector จัดการตัวเองได้) แต่โจทย์ข้อนี้ต้องการฝึกเขียน
    // copy-and-swap idiom เองเพื่อความเข้าใจกลไกเบื้องหลัง (ในโค้ดจริงมักปล่อยให้ Rule of Zero ทำงานแทน)
    Playlist(const Playlist& other) : owner_(other.owner_), songs_(other.songs_) {}
    Playlist(Playlist&& other) noexcept : owner_(), songs_() { swap(*this, other); }

    Playlist& operator=(Playlist other) { // รับ by value -> ได้ copy หรือ move มาแล้วแต่ argument
        swap(*this, other);
        return *this;
    }

    friend void swap(Playlist& a, Playlist& b) noexcept {
        using std::swap;
        swap(a.owner_, b.owner_);
        swap(a.songs_, b.songs_);
    }

    void print() const {
        std::cout << owner_ << "'s playlist: ";
        for (const auto& s : songs_) std::cout << "[" << s << "] ";
        std::cout << '\n';
    }

private:
    std::string owner_;
    std::vector<std::string> songs_;
};

int main() {
    Playlist p1("Somchai", {"Song A", "Song B"});
    Playlist p2("Somsri", {"Song C"});

    p2 = p1;             // copy assignment ผ่าน copy-and-swap
    p1.print();
    p2.print();

    Playlist p3("Suda", {"Song D", "Song E", "Song F"});
    p2 = std::move(p3);   // move assignment ผ่าน copy-and-swap ตัวเดียวกัน (คนละ overload กันไม่ต้องเขียนเพิ่ม)
    p2.print();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex3_playlist_copy_swap.cpp -o ex3
./ex3
```

```
Somchai's playlist: [Song A] [Song B] 
Somchai's playlist: [Song A] [Song B] 
Suda's playlist: [Song D] [Song E] [Song F]
```

สังเกตว่า `operator=(Playlist other)` เพียงตัวเดียว รองรับทั้ง `p2 = p1` (copy) และ
`p2 = std::move(p3)` (move) โดยไม่ต้องเขียน `operator=` แยกกันสองฟังก์ชันเลย — compiler เป็น
ผู้เลือกเองว่าจะสร้างพารามิเตอร์ `other` ด้วย copy constructor หรือ move constructor ตามชนิดของ
อาร์กิวเมนต์ที่ส่งเข้ามา

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- แยกแยะ **lvalue** และ **rvalue** ได้อย่างถูกต้อง และเข้าใจว่าทำไมความแตกต่างนี้ถึงเป็นรากฐาน
  ของทุกฟีเจอร์ใน Part นี้
- เข้าใจปัญหาที่แท้จริงที่ **move semantics** แก้ไข: การ copy ข้อมูลจำนวนมากโดยไม่จำเป็นเมื่อ
  ส่งต่อหรือ return ทรัพยากรระหว่างฟังก์ชัน
- ใช้ **rvalue reference (`&&`)** เขียน overload ที่แยกแยะ lvalue/rvalue และเข้าใจว่าตัวแปรที่
  มีชื่อทุกตัวเป็น lvalue เสมอ แม้จะประกาศด้วยชนิด `&&` ก็ตาม
- เข้าใจอย่างถ่องแท้ว่า **`std::move` เป็นแค่การ cast** ไม่ได้ย้ายอะไรด้วยตัวมันเอง การย้ายจริง
  เกิดขึ้นในฟังก์ชันที่รับค่านั้นไปใช้ต่างหาก
- เขียน **move constructor/move assignment operator** เองสำหรับ class ที่ถือ raw pointer ได้
  อย่างถูกต้อง พร้อมเข้าใจว่าทำไม `noexcept` ถึงสำคัญมากต่อประสิทธิภาพของ `std::vector`
- เข้าใจ **Rule of Five** ที่ขยายจาก Rule of Three และรู้ว่าการเขียนแค่ Rule of Three ในโลกที่
  มี move semantics จะทำให้เกิดการ fallback ไปเป็น copy อย่างเงียบๆ โดยไม่มีคำเตือนใดๆ
- ใช้ **`std::swap`** และ **copy-and-swap idiom** เขียน `operator=` ตัวเดียวที่รองรับได้ทั้ง
  copy และ move พร้อมได้ strong exception guarantee ฟรี
- เข้าใจแนวคิดเบื้องต้นของ **perfect forwarding** ผ่าน `std::forward` และ **forwarding
  reference** เพื่อเขียนฟังก์ชัน wrapper ที่ส่งต่ออาร์กิวเมนต์โดยไม่เสียข้อมูลว่าเป็น
  lvalue/rvalue

Move semantics คือฟีเจอร์เดียวของ C++11 ที่ส่งผลกระทบต่อการออกแบบโค้ดทั้งระบบมากที่สุด — ตั้งแต่
`std::vector`, `std::string`, ไปจนถึง `unique_ptr` ที่เรียนใน Part 67 ล้วนพึ่งพากลไกนี้เป็น
รากฐานทั้งสิ้น ใน **Part 71** เราจะเปลี่ยนโฟกัสไปที่อีกด้านหนึ่งของ "ประสิทธิภาพ" ใน Modern
C++: การคำนวณบางอย่างให้เกิดขึ้น **ที่ compile-time แทนที่จะเป็น runtime** ผ่านคีย์เวิร์ด
`constexpr` และ `consteval`

**ต่อไป:** [Part 71 — constexpr และ Compile-Time Programming](./part-071-constexpr.md)
