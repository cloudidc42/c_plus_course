# Part 79: CRTP และ Template Pattern ขั้นสูง (Step 625–632)

> Module F — Modern C++ (C++11 ถึง C++23) | Part 79 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 625–632
> Part ก่อนหน้า: [Part 78 — Metaprogramming ด้วย Template](./part-078-metaprogramming.md) | Part ถัดไป: [Part 80 — Modern C++ Best Practice และ C++ Core Guidelines](./part-080-modern-cpp-best-practices.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **CRTP (Curiously Recurring Template Pattern)** คืออะไร และทำไมชื่อของมันถึง
   "ประหลาด" (Curiously Recurring) ตามที่ James Coplien ตั้งชื่อไว้ในปี 1995
2. เขียน **Static Interface** ด้วย CRTP เพื่อจำลอง Polymorphism โดยไม่ต้องใช้ `virtual` function
3. อธิบายความแตกต่างระหว่าง **Static Polymorphism** (แก้ที่ compile time ผ่าน CRTP) กับ
   **Dynamic Polymorphism** (แก้ที่ runtime ผ่าน `virtual`/vtable จาก Part 49) ทั้งด้านกลไกภายในและ
   Performance จริงที่วัดได้
4. เขียน **Mixin Pattern** ด้วย CRTP เพื่อเพิ่มความสามารถ (เช่น comparison operators) ให้ class
   หลายตัวโดยไม่ต้องเขียนโค้ดซ้ำ
5. ใช้ CRTP ทำ Object Counter และรู้จักการใช้งานจริงของ CRTP ในไลบรารีมาตรฐาน
   (`std::enable_shared_from_this`)
6. วิเคราะห์ Error Message ที่เกิดจากการใช้ CRTP ผิดพลาด และรู้สาเหตุที่แท้จริง
7. ตัดสินใจได้ว่าเมื่อไหร่ควรใช้ CRTP และเมื่อไหร่ที่ความซับซ้อนของมัน "ไม่คุ้ม" กับสิ่งที่ได้มา

---

## 79.1 CRTP คืออะไร (Step 625)

**CRTP (Curiously Recurring Template Pattern)** คือรูปแบบการเขียนโค้ดที่ class ลูก (Derived)
สืบทอดจาก class แม่ (Base) ที่เป็น **Template ซึ่งรับตัวมันเองเป็น Template Argument**:

```cpp
template <typename Derived>
class Base { /* ... */ };

class MyClass : public Base<MyClass> {  // ส่งตัวเองเข้าไปเป็น Template Argument!
    // ...
};
```

ประโยคนี้อ่านครั้งแรกจะรู้สึกแปลกมาก — `MyClass` ยังไม่ถูกนิยามจบเลยด้วยซ้ำ (compiler กำลังอ่าน
`class MyClass : ...` อยู่) แต่ดันเอาชื่อ `MyClass` ไปใช้เป็น Template Argument ของ Base Class
ของตัวเองซะแล้ว! นี่คือที่มาของคำว่า **"Curiously Recurring"** (การเวียนกลับที่ชวนพิศวง) ที่
James Coplien ตั้งชื่อไว้ในบทความปี 1995

**ทำไมถึงทำแบบนี้ได้?** เพราะ ณ จุดที่คอมไพเลอร์เจอ `class MyClass : public Base<MyClass>`
มันแค่ต้องการ**ชื่อ** `MyClass` เพื่อใช้เป็น Template Argument (การประกาศชื่อ class เกิดขึ้นก่อน
เนื้อหาข้างในเสมอ) — ไม่ได้ต้องการรู้ **รายละเอียดภายใน** ของ `MyClass` ในขั้นตอนนี้ (Incomplete
Type ก็เพียงพอสำหรับใช้เป็น Template Argument ณ จุดนี้) รายละเอียดภายใน (เช่น member function
ต่างๆ) จะถูกใช้จริงก็ต่อเมื่อโค้ดของ `Base<MyClass>` ถูก instantiate และเรียกใช้งานจริงอีกที
ซึ่งเกิดขึ้นหลังจาก `MyClass` ถูกนิยามครบสมบูรณ์แล้ว

ลองดูตัวอย่างที่ใช้งานได้จริงตัวแรก:

```cpp
#include <iostream>

template <typename Derived>
class Shape {
public:
    double area() const {
        return static_cast<const Derived*>(this)->area_impl();
    }
    void print_area() const {
        std::cout << "พื้นที่ = " << area() << "\n";
    }
};

class Circle : public Shape<Circle> {
public:
    explicit Circle(double r) : radius_(r) {}
    double area_impl() const { return 3.14159265358979 * radius_ * radius_; }

private:
    double radius_;
};

class Square : public Shape<Square> {
public:
    explicit Square(double s) : side_(s) {}
    double area_impl() const { return side_ * side_; }

private:
    double side_;
};

int main() {
    Circle c(2.0);
    Square s(3.0);
    c.print_area();
    s.print_area();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 crtp_basic.cpp -o crtp_basic && ./crtp_basic
```

ผลลัพธ์:

```
พื้นที่ = 12.5664
พื้นที่ = 9
```

**กลไกสำคัญคือ `static_cast<const Derived*>(this)`** — เมื่อ `Shape<Circle>::area()` ถูกเรียกผ่าน
object ของ `Circle` จริง `this` (ที่เป็น `const Shape<Circle>*`) จะถูก cast กลับไปเป็น
`const Circle*` ทำให้เราสามารถเรียก `area_impl()` ที่นิยามอยู่ใน `Circle` ได้ตรงๆ — นี่คือการ
"เรียก method ของ derived class จาก base class" **โดยไม่ใช้ `virtual` เลยแม้แต่ตัวเดียว**

การ cast นี้ปลอดภัยเพราะเรารู้ล่วงหน้าจากโครงสร้าง Template ว่า `this` ที่ถูกส่งเข้ามาต้องเป็น
pointer ของ object ที่แท้จริงเป็น `Derived` เสมอ (เพราะ `Circle` สืบทอดจาก `Shape<Circle>` เท่านั้น
ไม่มีทางที่ object อื่นจะเรียก method นี้ผ่าน `Shape<Circle>` ได้)

---

## 79.2 Static Polymorphism vs Dynamic Polymorphism (Step 626)

รูปแบบที่เราเพิ่งเขียนใน 79.1 มีชื่อเรียกว่า **Static Polymorphism** ("Compile-Time Polymorphism")
เพื่อเปรียบเทียบกับ **Dynamic Polymorphism** ที่เราเรียนไปแล้วใน **Part 49** (Polymorphism และ
Virtual Function) ซึ่งใช้ `virtual` function และกลไก vtable/vptr

| ประเด็น | Dynamic Polymorphism (`virtual`, Part 49) | Static Polymorphism (CRTP) |
|---|---|---|
| ตัดสินใจเรียก method ไหน | ตอน **Runtime** ผ่าน vtable lookup | ตอน **Compile Time** ผ่าน `static_cast` |
| ต้องมี vtable/vptr ในแต่ละ object ไหม | ต้องมี (เพิ่มขนาด object 8 ไบต์ต่อ object บนระบบ 64-bit) | ไม่ต้องมีเลย — ไม่มี Overhead ด้าน memory |
| เก็บ object หลายชนิดใน container เดียวกันได้ไหม (`std::vector<Base*>`) | **ได้** — นี่คือจุดแข็งหลักของ Dynamic Polymorphism | **ไม่ได้โดยตรง** — แต่ละ `Base<T>` เป็นคนละ Type กันจริงๆ |
| Compiler สามารถ **Inline** การเรียกได้ไหม | ยากมาก (compiler ไม่รู้ล่วงหน้าว่า object จริงเป็นชนิดไหน) | **ได้ง่าย** — compiler รู้ชนิดที่แน่นอนตั้งแต่ compile time |
| ความยืดหยุ่น (เพิ่ม class ใหม่โดยไม่แก้ code เดิม) | สูง — Plugin/Runtime Registration ทำได้เป็นธรรมชาติ | ต่ำกว่า — ทุกอย่างต้องรู้ตอน compile |
| Error Message เมื่อเขียนผิด | สั้น ชัดเจน | มักยาวและอ่านยาก (ดู 79.6) |
| ใช้บ่อยในสถานการณ์ | Plugin system, GUI framework, Dependency Injection | Library ที่ต้องการ Performance สูงสุด (game engine, embedded, numerical library) |

**ทำไม CRTP ถึงเร็วกว่า?** เพราะการเรียก `virtual` function ทุกครั้งต้อง (1) อ่านค่า vptr จาก
object (2) กระโดดไปที่ vtable ของ class นั้น (3) หาตำแหน่งของฟังก์ชันใน vtable แล้วค่อยกระโดดไป
เรียกจริง — ทั้งหมดนี้เกิดขึ้น **ตอนรันจริง** ทุกครั้งที่เรียก และที่สำคัญกว่านั้นคือ **compiler
แทบไม่มีทาง Inline การเรียก virtual function ได้เลย** เพราะไม่รู้ล่วงหน้าว่า object จริงคือ class
ลูกตัวไหน (ยกเว้นกรณีพิเศษที่เรียกว่า Devirtualization ซึ่งทำได้จำกัดมาก)

ในขณะที่ CRTP ทุกอย่างถูกตัดสินใจตอน compile time — `static_cast<const Circle*>(this)->area_impl()`
คอมไพเลอร์รู้ชัดเจนตั้งแต่แรกว่ากำลังเรียก `Circle::area_impl()` ทำให้มันสามารถ **Inline** โค้ดทั้ง
ฟังก์ชันเข้าไปตรงจุดเรียกได้เลย ไม่มีการกระโดด ไม่มีการอ่าน vtable ใดๆ ทั้งสิ้น

---

## 79.3 วัด Performance จริง: CRTP vs Virtual Function (Step 627–628)

พูดลอยๆ ว่า "CRTP เร็วกว่า" ไม่พอ เรามาวัดจริงกันด้วยโค้ดที่รันซ้ำหลายล้านครั้งเพื่อให้เห็นความ
แตกต่างชัดเจน:

```cpp
#include <chrono>
#include <iostream>
#include <memory>
#include <vector>

// ---------- Dynamic polymorphism (virtual function) ----------
class ShapeBase {
public:
    virtual ~ShapeBase() = default;
    virtual double area() const = 0;
};

class DynCircle : public ShapeBase {
public:
    explicit DynCircle(double r) : radius_(r) {}
    double area() const override { return 3.14159265358979 * radius_ * radius_; }

private:
    double radius_;
};

// ---------- Static polymorphism (CRTP) ----------
template <typename Derived>
class StaticShape {
public:
    double area() const { return static_cast<const Derived*>(this)->area_impl(); }
};

class StaticCircle : public StaticShape<StaticCircle> {
public:
    explicit StaticCircle(double r) : radius_(r) {}
    double area_impl() const { return 3.14159265358979 * radius_ * radius_; }

private:
    double radius_;
};

int main() {
    constexpr int kIterations = 20'000'000;

    std::vector<std::unique_ptr<ShapeBase>> dyn_shapes;
    dyn_shapes.push_back(std::make_unique<DynCircle>(2.0));

    double dyn_sum = 0.0;
    auto t1 = std::chrono::steady_clock::now();
    for (int i = 0; i < kIterations; ++i) {
        dyn_sum += dyn_shapes[0]->area();
    }
    auto t2 = std::chrono::steady_clock::now();

    StaticCircle static_circle(2.0);
    double static_sum = 0.0;
    auto t3 = std::chrono::steady_clock::now();
    for (int i = 0; i < kIterations; ++i) {
        static_sum += static_circle.area();
    }
    auto t4 = std::chrono::steady_clock::now();

    const std::chrono::duration<double, std::milli> dyn_ms = t2 - t1;
    const std::chrono::duration<double, std::milli> static_ms = t4 - t3;

    std::cout << "Dynamic (virtual) sum = " << dyn_sum
              << ", เวลา = " << dyn_ms.count() << " ms\n";
    std::cout << "Static (CRTP)    sum = " << static_sum
              << ", เวลา = " << static_ms.count() << " ms\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 -O2 perf_compare.cpp -o perf_compare
./perf_compare
```

ผลลัพธ์ (วัดบนเครื่องทดสอบจริง ด้วย GCC 13.3 บน Linux, flag `-O2`):

```
Dynamic (virtual) sum = 2.51327e+08, เวลา = 56.6232 ms
Static (CRTP)    sum = 2.51327e+08, เวลา = 31.8664 ms
```

CRTP เร็วกว่าประมาณ **1.8 เท่า** ในการทดสอบนี้ (ตัวเลขจริงจะแตกต่างกันไปตามเครื่อง, compiler, และ
เวอร์ชันของ optimizer — **ควรวัดในสภาพแวดล้อมจริงของโปรเจกต์ตัวเองเสมอ** ไม่ใช่เชื่อตัวเลขจาก
บทความใดบทความหนึ่งตรงๆ) เหตุผลที่ตัวเลขต่างกันชัดเจนขนาดนี้เพราะ `-O2` ทำให้ compiler สามารถ
Inline การเรียก `static_circle.area()` เข้าไปในลูปได้ทั้งหมด กลายเป็นแค่การคำนวณเลขล้วนๆ ในขณะที่
การเรียกผ่าน `dyn_shapes[0]->area()` ยังต้องอ่าน vtable ทุกรอบเพราะ compiler ไม่สามารถพิสูจน์ได้ว่า
`dyn_shapes[0]` จะไม่ถูกเปลี่ยนชนิดระหว่างการวนลูป

> **หมายเหตุสำคัญ**: การเปรียบเทียบนี้เป็นตัวอย่างเพื่อการศึกษา ไม่ใช่ Benchmark ที่เข้มงวดตาม
> มาตรฐาน (เช่น ไม่มีการทำ Warm-up, ไม่ได้ป้องกัน CPU Frequency Scaling) การ Benchmark อย่างถูกต้อง
> และแม่นยำในระดับ Production จะเรียนอย่างละเอียดใน **Part 90 (Google Benchmark)**

**ในหลายกรณี ความแตกต่างด้าน Performance ระหว่าง virtual กับ CRTP มีขนาดเล็กมากจนวัดไม่ออกเลย**
โดยเฉพาะเมื่อฟังก์ชันที่เรียกทำงานหนักกว่าการอ่าน vtable มาก (เช่น เรียก database, network I/O)
ในกรณีนั้น Overhead ของ virtual function แทบไม่มีนัยสำคัญเลย — กฎทองคือ **อย่าเลือก CRTP เพียงเพราะ
"คิดว่า" มันเร็วกว่า ต้องวัดจริงในบริบทของโปรแกรมตัวเองก่อนเสมอ**

---

## 79.4 Mixin Pattern ด้วย CRTP (Step 629)

การใช้งาน CRTP ที่ได้รับความนิยมมากอีกแบบหนึ่งคือ **Mixin Pattern** — การเขียน "ความสามารถสำเร็จรูป"
ที่ class ไหนก็ตามสามารถ "ผสม" (mix in) เข้าไปได้ด้วยการสืบทอดจาก Template Base เพียงบรรทัดเดียว

ตัวอย่างคลาสสิกที่สุดคือการเพิ่ม **Comparison Operator ที่เหลือทั้งหมด** ให้ class ที่มีแค่
`operator==` กับ `operator<` อยู่แล้ว:

```cpp
#include <iostream>

template <typename Derived>
class Comparable {
public:
    friend bool operator!=(const Derived& a, const Derived& b) { return !(a == b); }
    friend bool operator>(const Derived& a, const Derived& b) { return b < a; }
    friend bool operator<=(const Derived& a, const Derived& b) { return !(b < a); }
    friend bool operator>=(const Derived& a, const Derived& b) { return !(a < b); }
};

class Money : public Comparable<Money> {
public:
    explicit Money(long cents) : cents_(cents) {}

    friend bool operator==(const Money& a, const Money& b) { return a.cents_ == b.cents_; }
    friend bool operator<(const Money& a, const Money& b) { return a.cents_ < b.cents_; }

private:
    long cents_;
};

int main() {
    const Money a(100);
    const Money b(200);

    std::cout << std::boolalpha;
    std::cout << "a == b : " << (a == b) << "\n";
    std::cout << "a != b : " << (a != b) << "\n";
    std::cout << "a <  b : " << (a < b) << "\n";
    std::cout << "a <= b : " << (a <= b) << "\n";
    std::cout << "a >  b : " << (a > b) << "\n";
    std::cout << "a >= b : " << (a >= b) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 mixin_comparable.cpp -o mixin_comparable
./mixin_comparable
```

ผลลัพธ์:

```
a == b : false
a != b : true
a <  b : true
a <= b : true
a >  b : false
a >= b : false
```

**เกิดอะไรขึ้น:** `Money` เขียนแค่ `operator==` กับ `operator<` เท่านั้น (สอง operator ที่จำเป็น
น้อยที่สุดในการนิยาม "ความเท่ากัน" และ "ลำดับ") ส่วน `operator!=`, `>`, `<=`, `>=` ทั้งหมดถูก
"แจกให้ฟรี" ผ่าน `Comparable<Money>` ที่ `Money` สืบทอดมา โดยแต่ละ operator เหล่านี้ถูกนิยามในรูป
ของ `==` และ `<` เท่านั้น (`!=` คือ "ไม่ใช่ `==`", `>` คือ "สลับข้างของ `<`" เป็นต้น)

**นี่คือพลังของ Mixin**: ถ้ามี class อีก 10 ตัวที่ต้องการ Comparison Operator ครบชุด แต่ละ class
แค่สืบทอดจาก `Comparable<ClassName>` แล้วเขียนแค่ `==` กับ `<` เท่านั้น ไม่ต้องเขียน `!=`, `>`,
`<=`, `>=` ซ้ำๆ กันสิบครั้ง — โค้ดสั้นลง บั๊กน้อยลง เพราะ logic การเปรียบเทียบเขียนไว้ที่เดียว

> **หมายเหตุสำหรับโปรเจกต์ C++20 ขึ้นไป**: ตั้งแต่ C++20 เป็นต้นมา ภาษามี **Three-Way Comparison
> (`operator<=>`, "Spaceship Operator")** ที่ทำสิ่งเดียวกันได้ในตัวภาษาเองโดยตรง เพียงเขียน
> `auto operator<=>(const Money&) const = default;` (หรือ custom logic) คอมไพเลอร์จะสร้าง
> `<`, `>`, `<=`, `>=` ให้อัตโนมัติทั้งหมด ทำให้ในโค้ดใหม่ Mixin Pattern แบบนี้สำหรับ Comparison
> **ไม่จำเป็นอีกต่อไปแล้ว** — แต่หลักการ Mixin ด้วย CRTP ยังมีประโยชน์มากสำหรับความสามารถอื่นๆ ที่
> ภาษาไม่มี built-in ให้ (เช่น Object Counter ใน 79.5, Logging, Serialization Interface ฯลฯ)

### Mixin อีกตัวอย่าง: สร้าง Postfix Operator จาก Prefix Operator

อีกหนึ่ง Mixin คลาสสิกที่ไลบรารีอย่าง Boost.Operators ใช้มานานคือการให้ class แค่ implement
**Prefix Increment (`++obj`)** เอง แล้วให้ Mixin สร้าง **Postfix Increment (`obj++`)** ให้ฟรี
(เพราะ Postfix เขียนในรูปของ Prefix ได้เสมอ: "จำค่าเดิมไว้ก่อน แล้วค่อยเพิ่มค่าจริง"):

```cpp
#include <iostream>

template <typename Derived>
class Incrementable {
public:
    Derived operator++(int) {  // postfix: รับ int (dummy parameter) เพื่อแยกจาก prefix
        Derived temp(static_cast<Derived&>(*this));
        ++static_cast<Derived&>(*this);  // เรียก prefix operator++() ที่ Derived implement เอง
        return temp;
    }
};

class Counter : public Incrementable<Counter> {
public:
    using Incrementable<Counter>::operator++;  // ดึง postfix operator++(int) จาก Base เข้ามา

    explicit Counter(int value) : value_(value) {}
    Counter& operator++() {  // prefix - Derived implement เอง
        ++value_;
        return *this;
    }
    int value() const { return value_; }

private:
    int value_;
};

int main() {
    Counter c(5);
    Counter old = c++;  // ใช้ postfix ที่มาจาก Mixin
    std::cout << "old.value() = " << old.value() << "\n";
    std::cout << "c.value()   = " << c.value() << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 postfix_mixin.cpp -o postfix_mixin && ./postfix_mixin
```

ผลลัพธ์:

```
old.value() = 5
c.value()   = 6
```

**จุดที่ต้องระวังมากในตัวอย่างนี้คือบรรทัด `using Incrementable<Counter>::operator++;`** ถ้าลบ
บรรทัดนี้ออก โค้ดจะ **คอมไพล์ไม่ผ่าน** ด้วย error `no 'operator++(int)' declared for postfix '++'`
ทั้งที่ `Incrementable<Counter>` มี `operator++(int)` ประกาศไว้จริง! สาเหตุคือกฎ **Name Hiding**
ของ C++: เมื่อ `Counter` ประกาศ member function ชื่อ `operator++` ของตัวเอง (แม้จะเป็นคนละ
Signature กัน — prefix ไม่มี parameter ส่วน postfix มี `int`) การประกาศนี้จะ **บดบัง (hide)**
member function ชื่อเดียวกันทั้งหมดที่มาจาก Base Class โดยอัตโนมัติ ไม่ว่า Signature จะตรงกันหรือไม่
ก็ตาม การเติม `using Incrementable<Counter>::operator++;` คือการบอกคอมไพเลอร์อย่างชัดเจนว่า "ให้ดึง
`operator++` ทุกตัวจาก Base เข้ามาอยู่ใน scope ของ Derived ด้วย" ซึ่งเป็น pattern ที่ต้องจำให้ขึ้นใจ
ทุกครั้งที่ทำ CRTP Mixin ที่มี Base และ Derived ประกาศฟังก์ชันชื่อเดียวกัน (ดู Pitfall ข้อ 7)

---

## 79.5 CRTP ในการนับ Object และการใช้งานจริงในไลบรารีมาตรฐาน (Step 630)

อีกตัวอย่างที่ใช้งานได้จริงคือ **Object Counter Mixin** — เพิ่มความสามารถนับจำนวน object ที่ยังมี
ชีวิตอยู่ให้กับ class ใดก็ได้ โดยแต่ละ class ที่สืบทอดจะได้ตัวนับ**แยกกันเป็นอิสระ**จากกัน:

```cpp
#include <iostream>

template <typename Derived>
class Counter {
public:
    Counter() { ++count_; }
    Counter(const Counter&) { ++count_; }
    ~Counter() { --count_; }

    static int live_count() { return count_; }

private:
    static inline int count_ = 0;
};

class Widget : public Counter<Widget> {};
class Gadget : public Counter<Gadget> {};

int main() {
    {
        Widget w1;
        Widget w2;
        Gadget g1;
        std::cout << "Widget ที่ยังมีชีวิตอยู่: " << Widget::live_count() << "\n";
        std::cout << "Gadget ที่ยังมีชีวิตอยู่: " << Gadget::live_count() << "\n";
        (void)w1;
        (void)w2;
        (void)g1;
    }
    std::cout << "หลังออกจาก scope, Widget: " << Widget::live_count() << "\n";
    std::cout << "หลังออกจาก scope, Gadget: " << Gadget::live_count() << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 object_counter.cpp -o object_counter
./object_counter
```

ผลลัพธ์:

```
Widget ที่ยังมีชีวิตอยู่: 2
Gadget ที่ยังมีชีวิตอยู่: 1
หลังออกจาก scope, Widget: 0
หลังออกจาก scope, Gadget: 0
```

**ทำไม `Widget::live_count()` กับ `Gadget::live_count()` ถึงนับแยกกัน ทั้งที่ใช้ template เดียวกัน?**
เพราะ `Counter<Widget>` และ `Counter<Gadget>` เป็นการ Instantiate Template **คนละครั้งกัน** จึงเป็น
**class คนละตัวกันโดยสมบูรณ์** (compiler สร้างโค้ดแยกกันสองชุด) ตัวแปร `static inline int count_`
ของแต่ละ instantiation จึงเป็นคนละตัวแปรกัน นี่คือจุดที่ CRTP ต่างจาก Base Class ธรรมดาที่ไม่ใช่
Template — ถ้า `Counter` เป็น class ปกติไม่ใช่ template, `Widget` และ `Gadget` จะแชร์ `count_`
ตัวเดียวกัน ซึ่งไม่ใช่สิ่งที่เราต้องการ

### การใช้งานจริงใน C++ Standard Library

CRTP ไม่ใช่แค่เทคนิคทฤษฎี มันถูกใช้จริงในระดับ Standard Library เอง ตัวอย่างที่ชัดเจนที่สุดคือ
`std::enable_shared_from_this<T>` (ที่เราจะเจอในรายละเอียดตอนทำงานกับ `shared_ptr` ขั้นสูง):

```cpp
class MyResource : public std::enable_shared_from_this<MyResource> {
public:
    std::shared_ptr<MyResource> get_shared() { return shared_from_this(); }
};
```

`std::enable_shared_from_this<MyResource>` ใช้โครงสร้าง CRTP เป๊ะๆ เพื่อให้ object ที่ถูกจัดการโดย
`shared_ptr` อยู่แล้ว สามารถสร้าง `shared_ptr` ตัวใหม่ที่ชี้มาที่ตัวเองได้อย่างปลอดภัย (แชร์
Control Block เดียวกัน ไม่สร้าง Control Block ซ้ำซ้อนที่จะนำไปสู่ Double Free) — นี่คือหลักฐานว่า
CRTP ไม่ใช่แค่ "ของเล่นทางทฤษฎี" แต่เป็นเครื่องมือที่ผู้ออกแบบภาษาเองเลือกใช้แก้ปัญหาจริง

---

## 79.6 ข้อจำกัดของ CRTP: Error Message ที่อ่านยาก (Step 631)

CRTP มีราคาที่ต้องจ่ายเสมอ และราคาที่ชัดเจนที่สุดคือ **Error Message ที่ซับซ้อนและอ่านยากกว่าโค้ด
ปกติมาก** เมื่อใช้งานผิดพลาด ลองดูความผิดพลาดที่พบบ่อยที่สุด — **ลืมส่ง Derived Class ที่ถูกต้อง
เข้าไปเป็น Template Argument**:

```cpp
#include <iostream>

template <typename Derived>
class Base {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }
};

// ผิด! ควรเป็น Base<Mistake> แต่ดันใส่ Base<int> ไป
class Mistake : public Base<int> {
public:
    void implementation() { std::cout << "implementation\n"; }
};

int main() {
    Mistake m;
    m.interface();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 broken_crtp.cpp -o broken_crtp
```

ผลลัพธ์ (Error จริงจาก GCC 13.3):

```
broken_crtp.cpp: In instantiation of 'void Base<Derived>::interface() [with Derived = int]':
broken_crtp.cpp:19:16:   required from here
broken_crtp.cpp:7:9: error: invalid 'static_cast' from type 'Base<int>*' to type 'int*'
    7 |         static_cast<Derived*>(this)->implementation();
      |         ^~~~~~~~~~~~~~~~~~~~~~~~~~~
```

สังเกตว่า Error พูดถึง `Base<Derived> [with Derived = int]` และ `invalid 'static_cast' from type
'Base<int>*' to type 'int*'` — ข้อความนี้ไม่ได้บอกตรงๆ ว่า **"คุณพิมพ์ `Base<int>` ผิด ที่ถูกคือ
`Base<Mistake>`"** ผู้เรียนต้องอนุมานเอาเองจากบริบทว่าปัญหาที่แท้จริงคือการสืบทอดผิดชนิด ซึ่งสำหรับ
มือใหม่ที่ไม่คุ้นกับ CRTP นี่คือ Error ที่ทำให้เสียเวลา Debug นานมาก

**ปัญหานี้จะรุนแรงขึ้นเรื่อยๆ** เมื่อ CRTP Base Class มีหลาย method, มี Template Parameter หลายตัว,
หรือถูกใช้ซ้อนกันหลายชั้น (CRTP บน CRTP) — Error Message อาจยาวเป็นหลายสิบหรือหลายร้อยบรรทัด และ
ชี้ไปที่ internal ของ Template แทนที่จะชี้ตรงจุดที่ผู้เขียนโค้ดทำผิดจริงๆ นี่คือเหตุผลสำคัญข้อหนึ่งที่
ทำให้ Concept (Part 73) ได้รับความนิยม — เพราะสามารถเขียน `static_assert` หรือ Concept กำกับไว้ที่
Base Class เพื่อให้ Error Message ชัดเจนขึ้นได้:

```cpp
#include <iostream>
#include <type_traits>

template <typename Derived>
class SafeBase {
public:
    SafeBase() {
        static_assert(std::is_base_of_v<SafeBase<Derived>, Derived>,
                      "Derived ต้องสืบทอดจาก SafeBase<Derived> เท่านั้น (CRTP ต้องใช้ตัวเองเป็น "
                      "Template Argument)");
    }
    void interface() { static_cast<Derived*>(this)->implementation(); }
};
```

การเติม `static_assert` แบบนี้ไว้ใน constructor ของ Base Class ทำให้ถ้ามีคนใช้ CRTP ผิด (เช่น
`class X : public SafeBase<Y>` โดย `Y` ไม่ใช่ `X`) จะได้ Error message ที่มนุษย์อ่านเข้าใจทันที
แทนที่จะต้องแกะ Template Instantiation Error ยาวๆ

---

## 79.7 เมื่อไหร่ไม่ควรใช้ CRTP (Step 632)

CRTP เป็นเครื่องมือที่ทรงพลังแต่ **ไม่ใช่ทางออกที่ถูกต้องเสมอไป** ต่อไปนี้คือสัญญาณที่บอกว่าควร
เลือกวิธีอื่นแทน:

| สถานการณ์ | เหตุผลที่ไม่ควรใช้ CRTP |
|---|---|
| ต้องเก็บ object หลายชนิดปนกันใน container เดียว (`std::vector<Base*>`) | CRTP ทำให้แต่ละ class เป็นคนละ Type กันจริง (`Base<A>` ต่างจาก `Base<B>`) ไม่มี Common Base ที่ใช้ container เดียวกันได้ตรงๆ ต้องใช้ Dynamic Polymorphism (Part 49) แทน |
| ระบบต้องรองรับ Plugin ที่โหลดตอน Runtime (เช่น `.so`/`.dll`) | CRTP ต้องรู้ชนิดข้อมูลทั้งหมดตอน Compile Time เข้ากันไม่ได้กับการโหลดโค้ดใหม่ตอนรันจริง |
| ทีมมีวิศวกรมือใหม่จำนวนมาก หรือ Codebase เน้นความเข้าใจง่ายเป็นหลัก | Error Message ที่ซับซ้อนของ CRTP (79.6) จะทำให้ทีมเสียเวลา Debug มากกว่าที่ประหยัดจาก Performance |
| ฟังก์ชันที่จะเรียกผ่าน Polymorphism ทำงานหนักอยู่แล้ว (I/O, Network, Database) | Overhead ของ `virtual` (การอ่าน vtable) เล็กน้อยมากเมื่อเทียบกับงานจริงที่ฟังก์ชันทำ — Optimize ผิดจุด (Premature Optimization) |
| ต้องการ Binary ที่ขนาดเล็ก (Embedded/Firmware ที่มี ROM จำกัด) | CRTP ทำให้ Compiler สร้างโค้ดแยกกันสำหรับทุก Derived Class ที่แตกต่างกัน (Code Bloat) ในขณะที่ Dynamic Polymorphism ใช้โค้ดชุดเดียวร่วมกันผ่าน vtable |
| ต้องเปลี่ยนพฤติกรรมของ object ตอน Runtime (เช่น Strategy Pattern ที่ต้องสลับ Algorithm ระหว่างการทำงาน) | Static Polymorphism ผูกพฤติกรรมไว้ตอน Compile Time แก้ไม่ได้แล้วตอนรัน ต้องใช้ Dynamic Polymorphism หรือ `std::function` แทน |

**กฎการตัดสินใจอย่างง่าย**: เริ่มต้นด้วย Dynamic Polymorphism (`virtual`) เสมอ เพราะเขียนง่ายกว่า
อ่านง่ายกว่า และ Error Message ชัดเจนกว่า — **ใช้ CRTP ก็ต่อเมื่อ** วัด Performance แล้วพบว่า
Overhead ของ `virtual` function เป็นคอขวดจริงๆ (Bottleneck ที่วัดได้ด้วยเครื่องมืออย่าง Part 86
Profiling) **และ** ไม่มีความจำเป็นต้องเก็บ object หลายชนิดปนกันใน container เดียว นี่คือหลักการ
"Optimize เมื่อมีหลักฐาน ไม่ใช่เมื่อมีความรู้สึก" ที่วิศวกรมืออาชีพทุกคนควรยึดถือ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมส่ง Derived Class ที่ถูกต้องเข้า Template Argument** — ดังตัวอย่างใน 79.6 การเขียน
   `class Mistake : public Base<int>` แทนที่จะเป็น `Base<Mistake>` ทำให้เกิด Error ที่อ่านยากมาก
   ป้องกันได้ด้วยการเติม `static_assert(std::is_base_of_v<Base<Derived>, Derived>, ...)` ใน Base

2. **เรียก method ของ Derived ก่อนที่ Derived Class จะถูกนิยามสมบูรณ์** — เนื่องจาก CRTP Base Class
   ถูก instantiate ตอนที่ Derived Class ยังไม่สมบูรณ์ (Incomplete Type) การพยายามใช้ `sizeof(Derived)`
   หรือเรียก non-static member function ของ `Derived` ใน **constructor ของ Base** อาจนำไปสู่
   Undefined Behavior ได้ ควรเรียกใช้ความสามารถของ Derived เฉพาะใน member function ธรรมดา
   (ไม่ใช่ constructor) ของ Base เท่านั้น

3. **คาดหวังว่า `Counter<Widget>` กับ `Counter<Gadget>` จะแชร์ static data กัน** — ตามที่อธิบายใน
   79.5 แต่ละ Instantiation ของ Template คือ class คนละตัวกันโดยสมบูรณ์ static member จึงแยกกันเสมอ
   ถ้าต้องการนับรวมทุก class ต้องเขียนตัวนับแยกต่างหากที่ไม่ผูกกับ CRTP

4. **ใช้ CRTP ทั้งที่ต้องการเก็บ object หลายชนิดใน container เดียว** — เป็นความผิดพลาดเชิง
   สถาปัตยกรรมที่พบบ่อยที่สุด: เขียน CRTP ไปแล้วครึ่งโปรเจกต์ถึงเพิ่งรู้ว่าต้องการ
   `std::vector<Shape*>` ที่เก็บทั้ง `Circle` และ `Square` ปนกัน ซึ่ง CRTP ทำไม่ได้ ต้องออกแบบใหม่
   เป็น Dynamic Polymorphism — ควรคิดเรื่องนี้ **ก่อน** เลือกใช้ CRTP เสมอ (ดูตาราง 79.7)

5. **สร้าง CRTP ที่ซ้อนกันหลายชั้นจนอ่านไม่ออก** — CRTP บน CRTP (เช่น `class A : public
   Mixin1<A>, public Mixin2<A>`) ใช้งานได้จริงและมีประโยชน์ (Multiple Mixin) แต่ถ้าซ้อนลึกเกินไป
   หรือ Mixin แต่ละตัวพึ่งพากันเอง Error Message จะซับซ้อนขึ้นแบบทวีคูณ ควรจำกัดความซับซ้อนและมี
   คอมเมนต์อธิบายความสัมพันธ์ไว้ชัดเจน

6. **ลืมว่า CRTP Base ที่มี member function จำเป็นต้องให้ Derived implement ทุกตัว ไม่งั้น
   Error จะไปโผล่ตอนเรียกใช้งานจริง ไม่ใช่ตอนประกาศ class** — ต่างจาก `virtual` function ล้วน
   (Pure Virtual) ที่ compiler บังคับ ณ จุดประกาศ class เลยว่าต้อง override ให้ครบ CRTP ไม่มีกลไก
   บังคับแบบนี้ในตัวเอง (นอกจาก concept กำกับด้วยตัวเอง) ทำให้บั๊ก "ลืม implement" มักถูกพบช้ากว่า

7. **ลืมเขียน `using Base<Derived>::member;` เมื่อ Derived ประกาศฟังก์ชันชื่อเดียวกับ Base (Name
   Hiding)** — ตามที่เห็นใน 79.4 (Postfix Mixin) การที่ `Derived` ประกาศ member function ชื่อ
   เดียวกับที่มาจาก `Base<Derived>` (แม้ Signature จะต่างกัน) จะบดบัง overload ทั้งหมดจาก Base
   จนกว่าจะมี `using` ประกาศดึงกลับเข้ามาอย่างชัดเจน นี่เป็นกฎทั่วไปของ Inheritance ใน C++ ไม่ใช่
   กฎเฉพาะของ CRTP แต่พบบ่อยมากเมื่อทำ Mixin ที่มีฟังก์ชันชื่อคล้ายกับที่ Derived ต้อง implement เอง

---

## แบบฝึกหัดท้ายบท

1. เขียน CRTP Base Class ชื่อ `Printable<Derived>` ที่มี method `void print() const` ซึ่งเรียก
   `to_display_string()` ของ Derived แล้วพิมพ์ออกทาง `std::cout` ทดสอบกับสอง class ที่แตกต่างกัน

2. ขยายตัวอย่าง `Comparable<Derived>` ใน 79.4 ให้เป็น class ชื่อ `Temperature` ที่เก็บอุณหภูมิเป็น
   `double` แล้วทดสอบ operator ทั้ง 6 ตัว (`==`, `!=`, `<`, `<=`, `>`, `>=`)

3. เขียน CRTP Mixin ชื่อ `Clonable<Derived>` ที่มี method `std::unique_ptr<Derived> clone() const`
   คืนค่าเป็นสำเนาใหม่ของ object (ใช้ Copy Constructor ของ `Derived`)

4. เขียน `SafeBase` ที่มี `static_assert` ตรวจสอบ CRTP ให้ถูกต้อง (ตามตัวอย่างใน 79.6) แล้วทดลอง
   สืบทอดผิดชนิดโดยตั้งใจ (comment เก็บ error message จริงที่ได้ไว้เป็นหลักฐาน)

5. เขียนโปรแกรมเปรียบเทียบ **ขนาดของ object** (`sizeof`) ระหว่าง class ที่สืบทอดจาก CRTP Base
   (ไม่มี virtual function เลย) กับ class ที่สืบทอดจาก Abstract Base ที่มี virtual function
   อธิบายว่าทำไมขนาดถึงต่างกัน (เกี่ยวข้องกับ vptr)

6. อธิบายด้วยคำพูดตัวเอง (เขียนเป็น comment) ว่าทำไม `std::vector<Shape<Circle>*>` ถึงเก็บ
   `Square` (ที่สืบทอดจาก `Shape<Square>`) ไม่ได้ ทั้งที่ `Circle` และ `Square` ดู "เหมือน" จะเป็น
   Shape เหมือนกัน

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <string>

template <typename Derived>
class Printable {
public:
    void print() const {
        std::cout << "[Printable] " << static_cast<const Derived*>(this)->to_display_string()
                  << "\n";
    }
};

class Point : public Printable<Point> {
public:
    Point(int x, int y) : x_(x), y_(y) {}
    std::string to_display_string() const {
        return "Point(" + std::to_string(x_) + ", " + std::to_string(y_) + ")";
    }

private:
    int x_;
    int y_;
};

class Temperature : public Printable<Temperature> {
public:
    explicit Temperature(double celsius) : celsius_(celsius) {}
    std::string to_display_string() const {
        return std::to_string(celsius_) + " C";
    }

private:
    double celsius_;
};

int main() {
    Point p(3, 4);
    Temperature t(36.6);
    p.print();
    t.print();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex1.cpp -o ex1 && ./ex1
```

ผลลัพธ์:

```
[Printable] Point(3, 4)
[Printable] 36.600000 C
```

**อธิบาย**: `Printable<Derived>::print()` ไม่รู้จักและไม่สนใจว่า `Derived` คืออะไร มันแค่เรียก
`to_display_string()` ผ่าน `static_cast` เหมือนใน 79.1 ทุกประการ — ทั้ง `Point` และ `Temperature`
ใช้ Mixin เดียวกันได้โดยแค่ implement `to_display_string()` ของตัวเอง ไม่ต้องเขียน logic การพิมพ์ซ้ำ

### แนวทางเฉลยข้อ 2

```cpp
#include <iostream>

template <typename Derived>
class Comparable {
public:
    friend bool operator!=(const Derived& a, const Derived& b) { return !(a == b); }
    friend bool operator>(const Derived& a, const Derived& b) { return b < a; }
    friend bool operator<=(const Derived& a, const Derived& b) { return !(b < a); }
    friend bool operator>=(const Derived& a, const Derived& b) { return !(a < b); }
};

class Temperature : public Comparable<Temperature> {
public:
    explicit Temperature(double celsius) : celsius_(celsius) {}

    friend bool operator==(const Temperature& a, const Temperature& b) {
        return a.celsius_ == b.celsius_;
    }
    friend bool operator<(const Temperature& a, const Temperature& b) {
        return a.celsius_ < b.celsius_;
    }

    double celsius() const { return celsius_; }

private:
    double celsius_;
};

int main() {
    const Temperature t1(25.0);
    const Temperature t2(30.0);

    std::cout << std::boolalpha;
    std::cout << "t1 == t2: " << (t1 == t2) << "\n";
    std::cout << "t1 != t2: " << (t1 != t2) << "\n";
    std::cout << "t1 <  t2: " << (t1 < t2) << "\n";
    std::cout << "t1 <= t2: " << (t1 <= t2) << "\n";
    std::cout << "t1 >  t2: " << (t1 > t2) << "\n";
    std::cout << "t1 >= t2: " << (t1 >= t2) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex2.cpp -o ex2 && ./ex2
```

ผลลัพธ์:

```
t1 == t2: false
t1 != t2: true
t1 <  t2: true
t1 <= t2: true
t1 >  t2: false
t1 >= t2: false
```

**อธิบาย**: `Temperature` เขียนแค่ `operator==` และ `operator<` เอง (จาก `celsius_` ที่เป็น
`double` เพียงตัวเดียว) ส่วนอีก 4 operator ที่เหลือได้มาฟรีจาก `Comparable<Temperature>` — โครงสร้าง
เดียวกับตัวอย่าง `Money` ใน 79.4 ทุกประการ เพียงเปลี่ยนชนิดข้อมูลภายในจาก `long` เป็น `double`

### แนวทางเฉลยข้อ 4

```cpp
#include <iostream>
#include <type_traits>

template <typename Derived>
class SafeBase {
public:
    SafeBase() {
        static_assert(std::is_base_of_v<SafeBase<Derived>, Derived>,
                      "Derived ต้องสืบทอดจาก SafeBase<Derived> เท่านั้น "
                      "(CRTP ต้องใช้ตัวเองเป็น Template Argument)");
    }
    void interface() { static_cast<Derived*>(this)->implementation(); }
};

class WrongUsage : public SafeBase<int> {  // ผิดโดยตั้งใจ เพื่อทดสอบ static_assert
public:
    void implementation() {}
};

int main() {
    WrongUsage w;
    w.interface();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex4.cpp -o ex4
```

Error message จริงที่ได้ (แสดงเฉพาะส่วนบนสุด — ส่วนสำคัญที่สุด):

```
ex4.cpp: In instantiation of 'SafeBase<Derived>::SafeBase() [with Derived = int]':
ex4.cpp:14:7:   required from here
ex4.cpp:8:28: error: static assertion failed: Derived ต้องสืบทอดจาก SafeBase<Derived> เท่านั้น
(CRTP ต้องใช้ตัวเองเป็น Template Argument)
ex4.cpp:8:28: note: 'std::is_base_of_v<SafeBase<int>, int>' evaluates to false
```

**อธิบาย**: เทียบกับ Error ดิบใน 79.6 ที่พูดแค่ `invalid 'static_cast' from type 'Base<int>*' to
type 'int*'` (ต้องอนุมานเอาเองว่าปัญหาคืออะไร) ข้อความจาก `static_assert` นี้บอก**ตรงๆ เป็น
ภาษาที่มนุษย์เขียนเอง** ว่าปัญหาคืออะไรและควรแก้อย่างไร พร้อม `note` ที่ยืนยันด้วยว่าเงื่อนไข
`is_base_of_v<SafeBase<int>, int>` เป็น `false` จริง — นี่คือคุณค่าที่แท้จริงของการเติม Guard
Clause แบบนี้ไว้ใน CRTP Base Class ทุกตัวที่จะให้คนอื่นในทีมนำไปใช้ต่อ

### แนวทางเฉลยข้อ 3

```cpp
#include <iostream>
#include <memory>
#include <string>

template <typename Derived>
class Clonable {
public:
    std::unique_ptr<Derived> clone() const {
        return std::make_unique<Derived>(static_cast<const Derived&>(*this));
    }
};

class Document : public Clonable<Document> {
public:
    explicit Document(std::string title) : title_(std::move(title)) {}
    const std::string& title() const { return title_; }
    void set_title(std::string title) { title_ = std::move(title); }

private:
    std::string title_;
};

int main() {
    Document original("รายงานประจำปี");
    auto copy = original.clone();
    copy->set_title("รายงานประจำปี (สำเนา)");

    std::cout << "ต้นฉบับ: " << original.title() << "\n";
    std::cout << "สำเนา:   " << copy->title() << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++20 ex3.cpp -o ex3 && ./ex3
```

ผลลัพธ์:

```
ต้นฉบับ: รายงานประจำปี
สำเนา:   รายงานประจำปี (สำเนา)
```

**อธิบาย**: `Clonable<Derived>::clone()` ใช้ `std::make_unique<Derived>` ร่วมกับ Copy Constructor
ของ `Derived` (ที่ compiler สร้างให้อัตโนมัติตาม Rule of Zero เพราะ `Document` มีแค่ `std::string`
เป็น member) การแก้ไข `copy` (เปลี่ยนชื่อเรื่อง) ไม่กระทบ `original` เลย เพราะเป็นการ Deep Copy
ผ่าน `std::string` ที่จัดการ memory ให้เองอย่างปลอดภัย

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจโครงสร้างและกลไกของ **CRTP (Curiously Recurring Template Pattern)** อย่างละเอียด ทั้งเหตุผล
  ที่ `static_cast<Derived*>(this)` ใช้งานได้อย่างปลอดภัย
- เปรียบเทียบ **Static Polymorphism (CRTP)** กับ **Dynamic Polymorphism (`virtual`, Part 49)**
  อย่างรอบด้าน ทั้งกลไกภายในและตารางเปรียบเทียบข้อดี-ข้อเสีย
- วัด Performance จริงระหว่างทั้งสองแนวทางด้วยโค้ดที่รันได้จริง และเข้าใจว่าทำไมผลลัพธ์ถึงต่างกัน
  (Inlining vs vtable lookup) พร้อมข้อควรระวังเรื่องการ Benchmark ที่ไม่เข้มงวด
- เขียน **Mixin Pattern** ด้วย CRTP เพื่อแจกความสามารถ (Comparison Operators) ให้หลาย class โดยไม่
  ต้องเขียนโค้ดซ้ำ พร้อมรู้จักทางเลือกใหม่ (`operator<=>`) ในโค้ด C++20
- เห็นการใช้งาน CRTP จริงในระดับ Standard Library ผ่าน `std::enable_shared_from_this`
- วิเคราะห์ Error Message จริงที่เกิดจากการใช้ CRTP ผิดพลาด และรู้วิธีป้องกันด้วย `static_assert`
- สร้างเกณฑ์ตัดสินใจที่ชัดเจนว่าเมื่อไหร่ควรและไม่ควรใช้ CRTP โดยยึดหลัก "Optimize เมื่อมีหลักฐาน"

ตลอด Module F เราได้เรียนรู้ฟีเจอร์และเทคนิคของ Modern C++ มาอย่างครบถ้วน ตั้งแต่ C++11 จนถึง C++23
รวมถึงเทคนิค Metaprogramming ขั้นสูงอย่าง SFINAE และ CRTP ใน **Part 80** ซึ่งเป็น Part สุดท้ายของ
Module นี้ เราจะ**สรุปทุกอย่างที่เรียนมาทั้งหมดตั้งแต่ Module D-F** ให้กลายเป็นแนวปฏิบัติที่ดี
(Best Practice) ที่ใช้ได้จริงในงานประจำวัน ผ่านมุมมองของ **C++ Core Guidelines** ซึ่งเป็นมาตรฐานที่
Bjarne Stroustrup และ Herb Sutter ร่วมกันวางไว้ให้วงการ C++ ทั้งหมด

**ต่อไป:** [Part 80 — Modern C++ Best Practice และ C++ Core Guidelines](./part-080-modern-cpp-best-practices.md)
