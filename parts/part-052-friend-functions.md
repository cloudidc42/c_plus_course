# Part 52: Friend Function และ Friend Class (Step 409–416)

> Module D — เริ่มต้น C++ และ OOP | Part 52 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 409–416
> Part ก่อนหน้า: [Part 51 — Operator Overloading](./part-051-operator-overloading.md) | Part ถัดไป: [Part 53 — Static Member และ Class Design](./part-053-static-members.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความหมายของ **Friend Function** และเขียน syntax การประกาศ friend ได้ถูกต้อง
2. อธิบายได้ว่าทำไมบางสถานการณ์ (เช่น `operator<<`) ถึงจำเป็นต้องใช้ friend function จริงๆ
3. อธิบายและใช้งาน **Friend Class** ได้ พร้อมเข้าใจว่ามันให้สิทธิ์อะไรบ้าง
4. เข้าใจคุณสมบัติพิเศษของ friend สามข้อ: **ไม่ symmetric, ไม่ transitive, ไม่ถูกสืบทอด**
5. วิเคราะห์ข้อถกเถียงเรื่อง "friend ทำลาย Encapsulation หรือไม่" ได้อย่างสมดุล ไม่เอียงไป
   ทางใดทางหนึ่งแบบสุดโต่ง
6. แยกแยะได้ว่าสถานการณ์ไหนที่ friend เป็น "ทางออกที่ดีที่สุด" และสถานการณ์ไหนที่การใช้
   friend เป็น **Anti-pattern** (ใช้เกินความจำเป็น)

---

## 52.1 Friend Function คืออะไร (Step 409)

Encapsulation ที่เรียนใน Part 47 สอนว่า `private` member ควรเข้าถึงได้จาก **member
function ของ class เดียวกันเท่านั้น** แต่ Part 51 ที่ผ่านมาเราเจอปัญหาสำคัญ: `operator<<`
สำหรับพิมพ์ object ผ่าน `std::cout` **ต้อง** เป็น non-member function (เพราะ operand ซ้าย
คือ `std::ostream` ไม่ใช่ class ของเรา) แต่บางครั้งฟังก์ชันนี้ก็จำเป็นต้องเข้าถึง private
data ของ class โดยตรง (เพื่อประสิทธิภาพ หรือเพราะไม่มี public getter ที่เหมาะสม)

C++ แก้ปัญหานี้ด้วยกลไกที่เรียกว่า **Friend Function** — class สามารถ "เชิญ" ฟังก์ชัน
ภายนอกที่ระบุชื่อไว้ชัดเจน ให้เข้าถึง private/protected member ของตัวเองได้ โดยประกาศ
คำว่า `friend` ไว้หน้า signature ของฟังก์ชันนั้น **ภายในตัว class**:

```cpp
#include <iostream>
#include <iomanip>

class Money {
public:
    Money(long baht, int satang = 0) : baht_(baht), satang_(satang) {}

    Money operator+(const Money& rhs) const {
        long total_satang = (baht_ * 100 + satang_) + (rhs.baht_ * 100 + rhs.satang_);
        return Money(total_satang / 100, static_cast<int>(total_satang % 100));
    }

    // ประกาศ friend: ให้ฟังก์ชันภายนอก (ไม่ใช่ member) เข้าถึง baht_/satang_ ที่เป็น private ได้
    // จำเป็นเพราะ operand ซ้ายของ << ต้องเป็น std::ostream ทำให้เขียนเป็น member function ไม่ได้
    friend std::ostream& operator<<(std::ostream& os, const Money& m);

private:
    long baht_;
    int satang_;
};

std::ostream& operator<<(std::ostream& os, const Money& m) {
    // ฟังก์ชันนี้ "ไม่ใช่" member ของ Money แต่เข้าถึง m.baht_ และ m.satang_ ได้โดยตรง
    // เพราะถูกประกาศเป็น friend ไว้ในคลาส
    os << m.baht_ << "." << std::setw(2) << std::setfill('0') << m.satang_ << " บาท";
    return os;
}
```

**สิ่งสำคัญที่ต้องเข้าใจให้แม่นยำ**:

1. `friend` ถูกประกาศ**ภายใน** class ที่ "ให้สิทธิ์" (ในที่นี้คือ `Money`) ไม่ใช่ภายใน
   ฟังก์ชันที่ "ได้รับสิทธิ์"
2. คำประกาศ `friend std::ostream& operator<<(...)` **ไม่ใช่** การประกาศ member function
   ของ `Money` — ฟังก์ชันนี้ยังคงเป็น non-member function ธรรมดา แค่ "ได้รับใบผ่านทาง"
   ให้เข้าไปดู private data ได้
3. Friend function ไม่มี `this` pointer ของ `Money` เลย มันต้องรับ parameter เป็น
   `const Money&` แล้วเข้าถึงผ่าน parameter นั้น (`m.baht_`) เหมือน non-member function
   ทั่วไปที่รับ object เข้ามา

```cpp
int main() {
    Money price1(120, 50);
    Money price2(79, 75);
    Money total = price1 + price2;

    std::cout << "price1 = " << price1 << '\n';
    std::cout << "price2 = " << price2 << '\n';
    std::cout << "total  = " << total << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 money.cpp -o money
./money
```

ผลลัพธ์:

```
price1 = 120.50 บาท
price2 = 79.75 บาท
total  = 200.25 บาท
```

---

## 52.2 ทำไมบางครั้งต้อง "จำเป็น" ใช้ Friend จริงๆ (Step 410)

คำถามที่ตามมาจากตัวอย่าง `Money` คือ: ทำไมไม่เขียน public getter อย่าง `baht()` และ
`satang()` แล้วให้ `operator<<` เรียกใช้แทนล่ะ? แบบที่เราทำกับ `Vector2D` ใน Part 51?

คำตอบคือ **ทำได้จริง และเป็นทางเลือกที่ดีกว่าในหลายกรณี** — ตัวอย่าง `Vector2D::operator<<`
ใน Part 51 ก็ไม่ได้ใช้ `friend` เลยเพราะใช้ public getter (`x()`, `y()`) เพียงพอแล้ว
คำถามที่แท้จริงคือ **friend จำเป็นเมื่อไหร่กันแน่?**

Friend function/class จำเป็นจริงๆ เมื่อเข้าเงื่อนไขข้อใดข้อหนึ่งต่อไปนี้:

1. **การเปิด public getter จะทำให้ class เสีย invariant หรือเปิดช่องให้ผู้ใช้ภายนอกแก้ไข
   ข้อมูลภายในโดยไม่ได้ตั้งใจ** เช่น class ที่เก็บ pointer ภายในที่ไม่ควรมีใครแก้ไขได้เลย
   นอกจากฟังก์ชันเฉพาะกลุ่มที่ไว้ใจได้
2. **ต้องการประสิทธิภาพสูงสุดโดยไม่ผ่านชั้น getter/setter** ในโค้ดที่ประสิทธิภาพสำคัญมาก
   (เช่น operator ที่ถูกเรียกนับล้านครั้งต่อวินาที) การเรียก getter ที่ inline ได้ดีอยู่แล้ว
   มักไม่ต่างกันในทางปฏิบัติ แต่ในบาง design ที่ getter มี logic ซับซ้อนกว่านั้น friend
   อาจช่วยได้จริง
3. **สอง class ทำงานร่วมกันแนบแน่นมากจนถือเป็น "หน่วยเดียวกัน" ในทางตรรกะ** (เช่น
   container กับ iterator ของมันเอง) การเปิด public interface ให้ครบทุกอย่างที่
   iterator ต้องการจะทำให้ container มี public surface ที่ใหญ่และรั่วไหลรายละเอียด
   ภายในออกไปโดยไม่จำเป็น
4. **ฟังก์ชัน/class ภายนอกต้องเข้าถึง private constructor** เพื่อจำกัดว่าใครสร้าง object
   ได้บ้าง (เทคนิคนี้เรียกว่า Factory Pattern ร่วมกับ friend ซึ่งจะเจอใน Part 97)

ตัวอย่าง `Money` เข้าเงื่อนไขข้อ 1 บางส่วน: สมมติว่าการออกแบบของทีมไม่ต้องการเปิด
`baht()`/`satang()` เป็น public เพราะไม่อยากให้ผู้ใช้ class คำนวณอะไรกับหน่วยเหล่านี้แยกกัน
เอง (อาจคำนวณผิดพลาดเรื่องทศนิยมได้ง่าย) อยากให้การเข้าถึงค่าดิบทำได้แค่จากภายในฟังก์ชัน
ที่ทีมเขียนเองและตรวจสอบแล้วเท่านั้น (เช่น `operator<<`, `operator+`) — นี่คือเหตุผลเชิง
การออกแบบที่สมเหตุสมผลสำหรับการใช้ `friend`

---

## 52.3 Friend Class (Step 411)

นอกจาก friend function แล้ว C++ ยังอนุญาตให้ friend เป็น **class ทั้งก้อน** ได้ด้วย —
เมื่อประกาศ `friend class X;` แปลว่า **ทุก member function ของ X** สามารถเข้าถึง
private/protected member ของ class ที่ประกาศไว้ได้ทั้งหมด

```cpp
#include <iostream>

class Engine {
public:
    explicit Engine(int horsepower) : horsepower_(horsepower) {}

    void display() const {
        std::cout << "Engine: " << horsepower_ << " แรงม้า, "
                  << "รอบเครื่องยนต์ = " << rpm_ << " RPM\n";
    }

private:
    // Car::start_engine() ต้องปรับ rpm_ โดยตรงเพราะเป็นส่วนหนึ่งของกลไก "สตาร์ทรถ" เดียวกัน
    // ทำให้ Engine ประกาศให้ Car เป็น friend class แทนที่จะเปิด public setter ให้ทุกคนเรียกได้
    int horsepower_;
    int rpm_ = 0;

    friend class Car;
};

class Car {
public:
    explicit Car(int horsepower) : engine_(horsepower) {}

    void start() {
        // เข้าถึง engine_.rpm_ ได้ตรงๆ เพราะ Car เป็น friend ของ Engine
        engine_.rpm_ = 800;
        std::cout << "รถสตาร์ทติดแล้ว\n";
        engine_.display();
    }

private:
    Engine engine_;
};

int main() {
    Car car(150);
    car.start();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 car.cpp -o car
./car
```

ผลลัพธ์:

```
รถสตาร์ทติดแล้ว
Engine: 150 แรงม้า, รอบเครื่องยนต์ = 800 RPM
```

ในตัวอย่างนี้ `rpm_` ไม่มี public setter เลย (ตั้งใจไม่ให้ใครแก้ไขค่านี้ตรงๆ จากภายนอก
เพราะมันควรถูกควบคุมโดยกลไกการสตาร์ทเครื่องยนต์เท่านั้น) แต่ `Car` ซึ่งเป็นส่วนหนึ่งของ
กลไกเดียวกัน (รถกับเครื่องยนต์ผูกกันแน่นจริงๆ ในทางตรรกะของโดเมนนี้) จำเป็นต้องเข้าถึง
`rpm_` โดยตรงเมื่อสตาร์ทเครื่อง การใช้ `friend class Car;` แทนการเปิด `set_rpm()` เป็น
public ทำให้ `rpm_` ยังคงถูกป้องกันจากโค้ดภายนอกอื่นๆ ทั้งหมด เหลือเพียง `Car` เท่านั้นที่
มีสิทธิ์พิเศษนี้

---

## 52.4 คุณสมบัติพิเศษของ Friend (Step 412)

Friend มีคุณสมบัติสำคัญ 3 ข้อที่มักทำให้ผู้เรียนสับสน เพราะมันต่างจากสัญชาตญาณที่คิดว่า
"friend น่าจะเป็นความสัมพันธ์สองทาง" หรือ "น่าจะส่งต่อกันได้เหมือน inheritance"

### 1. Friend ไม่ Symmetric (ไม่ใช่ความสัมพันธ์สองทาง)

การที่ `A` ประกาศ `friend class B;` หมายความว่า **`B` เข้าถึง private ของ `A` ได้**
แต่ **ไม่ได้แปลว่า `A` เข้าถึง private ของ `B` ได้กลับ** ถ้าต้องการให้เป็นสองทาง ต้อง
ประกาศ friend ทั้งสองฝั่งแยกกันเอง

```cpp
class Vault {
public:
    friend class Auditor;   // Vault ให้ Auditor เข้าถึง private ของตัวเองได้

private:
    int gold_ = 1000;
};

class Auditor {
public:
    static void inspect(const Vault& v) {
        // เข้าถึง v.gold_ ได้เพราะ Vault ประกาศ friend class Auditor ไว้
        std::cout << "ทองคำในคลัง: " << v.gold_ << '\n';
    }

private:
    int notes_ = 7;   // ข้อมูลส่วนตัวของ Auditor เอง

    // Vault ไม่ได้เป็น friend ของ Auditor (ไม่ได้ประกาศ friend class Vault; ไว้ที่นี่)
    // ดังนั้น Vault จะเข้าถึง Auditor::notes_ ไม่ได้เลย แม้ Auditor จะเข้าถึง Vault ได้ก็ตาม
};
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 not_symmetric.cpp -o not_symmetric
./not_symmetric
# ทองคำในคลัง: 1000
# ความสัมพันธ์แบบ friend ไม่ symmetric: Vault ให้สิทธิ์ Auditor แต่ Auditor ไม่ได้ให้สิทธิ์ Vault กลับ
```

### 2. Friend ไม่ Transitive (ไม่ส่งต่อกันเป็นทอดๆ)

ถ้า `A` เป็น friend ของ `B` และ `B` เป็น friend ของ `C` **ไม่ได้แปลว่า `A` เป็น friend
ของ `C` โดยอัตโนมัติ** ความไว้ใจใน C++ ไม่ส่งต่อกันเหมือนคำแนะนำจากเพื่อนของเพื่อน —
ถ้า `C` ต้องการให้ `A` เข้าถึง private ของตัวเองได้ ต้องประกาศ `friend class A;` ใน `C`
โดยตรงเท่านั้น ไม่มีทางลัดใดๆ

### 3. Friend ไม่ถูกสืบทอด (Not Inherited)

ถ้า `Base` ประกาศ `friend class Trusted;` แล้ว `Derived` สืบทอดจาก `Base` — `Trusted`
จะเข้าถึง**ส่วนที่ `Derived` รับช่วงมาจาก `Base`** ได้ (เพราะมันคือสมาชิกของ `Base` จริงๆ)
แต่ **จะเข้าถึง private member ใหม่ที่ `Derived` ประกาศเพิ่มเองไม่ได้เลย**

```cpp
class Base {
public:
    friend class Trusted;   // Trusted เป็น friend ของ Base เท่านั้น

private:
    int secret_ = 42;
};

class Derived : public Base {
private:
    int extra_ = 99;   // สมาชิกใหม่ของ Derived เอง (ไม่ได้มาจาก Base)
};

class Trusted {
public:
    static void reveal(const Base& b) {
        // เข้าถึง secret_ ได้เพราะ Trusted เป็น friend ของ Base โดยตรง
        std::cout << "Trusted เห็น Base::secret_ = " << b.secret_ << '\n';
    }

    // static void revealExtra(const Derived& d) {
    //     std::cout << d.extra_ << '\n';
    //     // ERROR: 'int Derived::extra_' is private within this context
    //     // เพราะ Trusted ไม่ใช่ friend ของ Derived (friend ไม่ถูกสืบทอด)
    // }
};

int main() {
    Base b;
    Trusted::reveal(b);

    Derived d;
    Trusted::reveal(d);   // ใช้ได้ เพราะ secret_ เป็นส่วนของ Base ที่ d รับช่วงมา

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 not_inherited.cpp -o not_inherited
./not_inherited
```

ผลลัพธ์:

```
Trusted เห็น Base::secret_ = 42
Trusted เห็น Base::secret_ = 42
```

ทั้งสามข้อนี้สรุปเป็นแนวคิดเดียวได้ว่า: **friend คือสิทธิ์ที่ผูกกับ "ชื่อ class/function
ที่ระบุไว้ตรงๆ" เท่านั้น ไม่ใช่แนวคิดที่ไหลไปตามความสัมพันธ์อื่นๆ ในระบบ** จึงต้องประกาศ
ให้ชัดเจนทุกครั้งที่ต้องการมอบสิทธิ์นี้

---

## 52.5 ข้อถกเถียง: Friend ทำลาย Encapsulation หรือไม่ (Step 413)

นี่เป็นหัวข้อที่ถกเถียงกันในวงการ C++ มานาน มีสองมุมมองสุดโต่งที่ควรรู้จักทั้งคู่ ก่อนจะ
สรุปมุมมองที่สมดุลกว่า:

**มุมมองที่ 1 (ต่อต้าน friend อย่างสิ้นเชิง)**: "friend ทำลาย Encapsulation เพราะเปิด
ช่องให้ class/function ภายนอกเข้าถึง private data ได้ ขัดกับหลักการ OOP พื้นฐานที่บอกว่า
'private ต้องเข้าถึงได้จากภายในเท่านั้น' ควรใช้ public getter/setter แทนเสมอ"

**มุมมองที่ 2 (สนับสนุน friend อย่างเต็มที่)**: "friend ไม่ได้ทำลาย Encapsulation เลย
เพราะการประกาศ friend ยังคงอยู่ **ภายใน** class ที่เป็นเจ้าของข้อมูล — class ยังคงเป็น
คนตัดสินใจว่าจะให้ใครเข้าถึง private ของตัวเองได้บ้าง (explicit consent) ไม่ใช่ใครก็ได้
เข้าถึงได้ตามใจชอบ (ซึ่งต่างจาก `public` โดยสิ้นเชิง)"

### มุมมองที่สมดุล (แนวทางของหลักสูตรนี้)

ความจริงอยู่ตรงกลางระหว่างสองมุมมองนี้:

| ประเด็น | คำอธิบาย |
|---|---|
| **Encapsulation ที่แท้จริงคืออะไร** | Encapsulation ไม่ได้หมายถึง "ห้ามใครเข้าถึง private เด็ดขาด" แต่หมายถึง "class ควบคุมได้ว่าใครเข้าถึงอะไรได้บ้าง" การประกาศ friend เป็นการควบคุมรูปแบบหนึ่ง (whitelist) ไม่ใช่การเปิดเผยทุกอย่างแบบ `public` |
| **friend ขยาย "ขอบเขต" ของ class ไม่ใช่ทำลายมัน** | มองอีกมุมหนึ่ง `friend` ทำให้ class + friend function/class ของมันกลายเป็น "หน่วยการออกแบบเดียวกัน" (unit of encapsulation) ที่ขยายออกไปเล็กน้อย ไม่ใช่การเปิด private ให้โลกภายนอกทั้งหมด |
| **ปัญหาจริงคือการใช้ friend มากเกินไป** | ถ้า class หนึ่งมี `friend` มากมายหลายสิบตัวโดยไม่มีเหตุผลชัดเจน นั่นคือสัญญาณของการออกแบบที่แย่ (ดู 52.7) แต่การมี friend หนึ่งหรือสองตัวที่มีเหตุผลชัดเจน (เช่น `operator<<`) ไม่ใช่ปัญหา |
| **ทางเลือกอื่นก็มีข้อเสียเช่นกัน** | การเปิด public getter ทุกตัวเพื่อหลีกเลี่ยง friend อาจทำให้ private data รั่วไหลออกไปสู่ **ทุกคน** ในระบบ ซึ่งแย่กว่าการให้สิทธิ์เฉพาะเจาะจงกับ friend ที่ระบุชื่อไว้ชัดเจนเสียอีก |

**สรุปเป็นแนวปฏิบัติที่ใช้ได้จริง**: ใช้ friend เมื่อมันทำให้ **การออกแบบโดยรวมง่ายขึ้นและ
ปลอดภัยกว่า** การเปิด public interface ให้กว้างขึ้น แต่ให้ตั้งคำถามกับตัวเองเสมอว่า "มี
ทางเลือกอื่นที่ไม่ต้องใช้ friend แล้วยังคงสะอาดเท่ากันไหม" ก่อนตัดสินใจใช้ friend ทุกครั้ง

---

## 52.6 ตัวอย่างจริงที่ Friend เป็นทางออกที่ดีที่สุด (Step 414)

มาดูตัวอย่างที่ friend เป็นทางออกที่เหมาะสมที่สุดจริงๆ: การสร้าง Iterator ของตัวเองสำหรับ
container ที่ออกแบบเอง Container กับ Iterator ของมันเป็น "หน่วยเดียวกัน" ในทางตรรกะ
อย่างชัดเจน — Iterator ต้องเดินผ่าน internal structure ของ container โดยตรงเพื่อ
ประสิทธิภาพสูงสุด แต่ container ก็ไม่ควรเปิด public getter คืน pointer ดิบให้ใครก็ได้ใน
ระบบใช้งานตามใจชอบ:

```cpp
#include <iostream>

class ListIterator;   // ประกาศล่วงหน้าเพื่อใช้เป็น friend ใน LinkedList

class LinkedList {
public:
    void push_back(int value) {
        Node* node = new Node{value, nullptr};
        if (!head_) {
            head_ = tail_ = node;
        } else {
            tail_->next = node;
            tail_ = node;
        }
    }

    ~LinkedList() {
        Node* current = head_;
        while (current) {
            Node* next = current->next;
            delete current;
            current = next;
        }
    }

    friend class ListIterator;

private:
    struct Node {
        int value;
        Node* next;
    };

    Node* head_ = nullptr;
    Node* tail_ = nullptr;
};

// ListIterator เป็นคนละ class กับ LinkedList แต่ต้องไล่ดู Node* ภายในโดยตรง
// เพื่อความเร็ว (ไม่อยากให้ LinkedList เปิด public getter คืน pointer ดิบออกมา)
// จึงขอเป็น friend class แทน
class ListIterator {
public:
    explicit ListIterator(const LinkedList& list) : current_(list.head_) {}

    bool has_next() const { return current_ != nullptr; }

    int next() {
        int value = current_->value;
        current_ = current_->next;   // เข้าถึง private struct Node ได้เพราะเป็น friend
        return value;
    }

private:
    LinkedList::Node* current_;
};

int main() {
    LinkedList list;
    list.push_back(10);
    list.push_back(20);
    list.push_back(30);

    ListIterator it(list);
    std::cout << "รายการ: ";
    while (it.has_next()) {
        std::cout << it.next() << " ";
    }
    std::cout << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 list_iterator.cpp -o list_iterator
./list_iterator
# รายการ: 10 20 30
```

สังเกตว่าถ้าไม่มี `friend class ListIterator;` เราจะต้องเปิด public method ให้
`LinkedList` คืน raw pointer ของ `head_` ออกไปให้ใครก็ได้เรียกใช้ (เช่น `Node*
get_head_unsafe()`) ซึ่งอันตรายกว่ามาก เพราะทุกโค้ดในระบบ (ไม่ใช่แค่ `ListIterator`)
จะสามารถแก้ไข linked list ผ่าน raw pointer นี้ได้โดยตรง ทำให้ invariant ของ `LinkedList`
(เช่น `tail_` ต้องชี้ไปยัง node สุดท้ายเสมอ) พังได้ง่ายมาก การใช้ `friend` ในกรณีนี้จึง
"ปลอดภัยกว่า" การเปิด public interface ให้กว้างขึ้น ตรงกับหลักการที่สรุปไว้ใน 52.5

STL เองก็ใช้แนวคิดคล้ายกันนี้ภายใน implementation ของ `std::vector::iterator`,
`std::map::iterator` ฯลฯ (แม้รายละเอียดการ implement จริงจะซับซ้อนกว่านี้มาก) หลักการ
พื้นฐานที่ container กับ iterator ผูกพันกันแน่นแฟ้นเป็นสิ่งที่เจอได้ทั่วไปในการออกแบบ
Data Structure

---

## 52.7 Anti-pattern: ใช้ Friend เกินความจำเป็น (Step 415)

มาดูตัวอย่างตรงข้ามกันบ้าง — สถานการณ์ที่ใช้ `friend` **โดยไม่จำเป็น** ซึ่งเป็นการออกแบบ
ที่แย่และควรหลีกเลี่ยง:

```cpp
// ANTI-PATTERN: ใช้ friend เกินความจำเป็น
// (คอมไพล์และรันได้ปกติ แต่เป็นตัวอย่างการออกแบบที่แย่)
#include <iostream>
#include <string>

class BankAccount {
public:
    BankAccount(std::string owner, double balance)
        : owner_(std::move(owner)), balance_(balance) {}

    // ปัญหา: ไม่มีเหตุผลอะไรเลยที่ AuditReport ต้อง "แหวก" เข้ามาแก้ balance_ ตรงๆ
    // ทั้งที่สามารถใช้ member function withdraw()/deposit() ที่มี validation ได้อยู่แล้ว
    // การเปิด friend แบบนี้ทำให้ AuditReport ผูกติดกับโครงสร้างภายในของ BankAccount
    // เปลี่ยนชื่อ balance_ หรือเปลี่ยน type วันหน้า โค้ด AuditReport จะพังทันที
    friend class AuditReport;

    void deposit(double amount) {
        if (amount > 0) {
            balance_ += amount;
        }
    }

    double balance() const { return balance_; }
    const std::string& owner() const { return owner_; }

private:
    std::string owner_;
    double balance_;
};

// class นี้ "ควร" ใช้แค่ getter อย่าง balance() ก็พอ แต่กลับเลือกแก้ private data ตรงๆ
class AuditReport {
public:
    static void apply_correction(BankAccount& acc, double correction) {
        acc.balance_ += correction;   // ควรเรียก acc.deposit(correction) แทน
        std::cout << "ปรับยอดบัญชีของ " << acc.owner_
                  << " เป็น " << acc.balance_ << " บาท (ผ่านการแก้ private ตรงๆ)\n";
    }
};

int main() {
    BankAccount acc("สมชาย", 1000.0);
    AuditReport::apply_correction(acc, 50.0);
    std::cout << "ยอดคงเหลือ: " << acc.balance() << " บาท\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 overuse.cpp -o overuse
./overuse
# ปรับยอดบัญชีของ สมชาย เป็น 1050 บาท (ผ่านการแก้ private ตรงๆ)
# ยอดคงเหลือ: 1050 บาท
```

### ทำไมนี่ถึงเป็น Anti-pattern

1. **มี public interface ที่เพียงพออยู่แล้ว** — `BankAccount` มี `deposit()` และ
   `balance()` เป็น public อยู่แล้ว `AuditReport` สามารถใช้ `acc.deposit(correction)`
   ได้เลยโดยไม่ต้องขอสิทธิ์ friend เพิ่มเลย
2. **ผูกโครงสร้างภายในเข้าด้วยกันโดยไม่จำเป็น** — ถ้าวันหนึ่งทีมพัฒนา `BankAccount`
   เปลี่ยนชื่อ `balance_` เป็น `balance_cents_` (เก็บเป็นหน่วยสตางค์แทนหน่วยบาท) โค้ดของ
   `AuditReport` จะพังทันทีเพราะมันรู้จักและพึ่งพารายละเอียดภายในตรงๆ ทั้งที่ควรจะพึ่งพา
   แค่ public interface ที่เสถียรกว่ามาก
3. **ข้ามผ่าน validation logic** — สมมติว่า `deposit()` มีการตรวจสอบ `amount > 0` ไว้
   (ป้องกันไม่ให้ฝากเงินติดลบ) การที่ `AuditReport::apply_correction` แก้ `balance_`
   ตรงๆ ทำให้ validation logic นี้ถูกข้ามไปโดยสิ้นเชิง เปิดช่องให้เกิด bug ที่ตรวจจับยาก
4. **ขยายจำนวน "โค้ดที่ต้องอ่านคู่กัน" เพื่อเข้าใจ class** — เมื่อมี friend หลายตัว คนที่
   ต้องการเข้าใจว่า `balance_` ถูกแก้ไขจากที่ไหนได้บ้าง ต้องไปอ่านโค้ดของทุก friend class
   ด้วย ไม่ใช่แค่ member function ของ `BankAccount` เอง ทำให้การไล่ตามโค้ด (traceability)
   ยากขึ้นมาก

**วิธีแก้ที่ถูกต้อง**: ลบ `friend class AuditReport;` ออก แล้วให้ `AuditReport` เรียก
`acc.deposit(correction)` แทนที่จะแก้ `balance_` ตรงๆ — โค้ดจะปลอดภัยกว่า ทดสอบง่ายกว่า
และไม่ผูกติดกับรายละเอียดภายในของ `BankAccount` เลย

---

## 52.8 แนวทางปฏิบัติ: เมื่อไหร่ควรใช้ Friend (Step 416)

สรุปเป็น checklist ที่ใช้ตัดสินใจได้จริงในงานออกแบบ:

| คำถามที่ควรถาม | ถ้าตอบว่า "ใช่" |
|---|---|
| มี public interface ที่เพียงพอสำหรับงานนี้อยู่แล้วหรือไม่? | **ไม่ต้องใช้ friend** — ใช้ public interface ที่มีอยู่ไปเลย |
| operand ซ้ายของ operator ที่ overload เป็น class อื่นที่ไม่ใช่ของเรา (เช่น `std::ostream`) หรือไม่? | ต้องเป็น non-member function — ใช้ friend **ถ้า** ไม่มี public getter เพียงพอ |
| สอง class นี้เป็น "หน่วยการออกแบบเดียวกัน" จริงๆ หรือไม่ (เช่น container กับ iterator ของมันเอง)? | ใช้ friend class ได้อย่างสมเหตุสมผล |
| การเปิด public getter/setter จะทำให้ invariant ของ class เสียหรือเปิดช่องให้แก้ไขข้อมูลผิดวิธีหรือไม่? | friend ที่จำกัดสิทธิ์เฉพาะบาง class อาจปลอดภัยกว่าการเปิด public ให้ทุกคน |
| จำนวน friend ใน class นี้เยอะขึ้นเรื่อยๆ โดยไม่มีเหตุผลชัดเจนแต่ละตัวหรือไม่? | สัญญาณอันตราย — ทบทวนการออกแบบใหม่ อาจต้องแยก responsibility ใหม่ทั้งหมด |

**กฎทองสั้นๆ ที่จำง่าย**: ใช้ `friend` เมื่อมันทำให้การออกแบบ **ปลอดภัยกว่าและชัดเจนกว่า**
การเปิด public interface ให้กว้างขึ้น ไม่ใช่ใช้เพราะ "ขี้เกียจเขียน getter" หรือ "ไม่อยาก
คิดเรื่อง access control ให้ดี" — ทุกครั้งที่เขียน `friend` ควรมีประโยคเหตุผลสั้นๆ กำกับไว้
เป็น comment เสมอ (ตามที่ทำในทุกตัวอย่างของ Part นี้) เพื่อให้คนอ่านโค้ดในอนาคต (รวมถึง
ตัวเราเองในอีกหกเดือนข้างหน้า) เข้าใจทันทีว่าทำไมความสัมพันธ์พิเศษนี้ถึงมีอยู่

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **คิดว่า `friend` เป็น member function ของ class ที่ประกาศมัน** — `friend std::ostream&
   operator<<(...)` ไม่ใช่ member function เลย มันยังคงเป็น non-member function ธรรมดา
   ที่แค่ "ได้รับสิทธิ์พิเศษ" การพยายามเรียกมันแบบ `money.operator<<(std::cout)` จะผิด
2. **คิดว่า friend เป็นความสัมพันธ์สองทาง** — ตามที่อธิบายใน 52.4 ถ้าต้องการให้ทั้งสอง
   class เข้าถึงกันและกันได้ ต้องประกาศ `friend` แยกกันทั้งสองฝั่ง
3. **คิดว่า friend ส่งต่อผ่านการสืบทอด** — `Derived` ไม่ได้รับสิทธิ์ friend ที่ `Base`
   มีมาโดยอัตโนมัติ และ friend ของ `Base` ก็เข้าถึงสมาชิกใหม่ที่ `Derived` เพิ่มเองไม่ได้
4. **ลืมประกาศ forward declaration เมื่อสอง class อ้างอิงกันไปมา** — เมื่อ `A` ต้อง
   `friend class B;` และ `B` มี member function ที่รับ `A` เป็น parameter (หรือกลับกัน)
   มักต้องมี forward declaration (`class B;`) ไว้ก่อน class ที่ใช้ชื่อนั้นเสมอ
5. **ใช้ friend แทนการออกแบบ public interface ที่ดี** — ตามที่เห็นใน 52.7 นี่คือ
   Anti-pattern ที่พบบ่อยที่สุด: ใช้ friend เป็นทางลัดเพื่อหลีกเลี่ยงการคิดออกแบบ
   public API ที่เหมาะสม ทำให้โค้ดผูกติดกันแน่นเกินจำเป็น (Tight Coupling)
6. **ใช้ friend มากเกินไปจนทำลายจุดประสงค์ของ `private`** — ถ้า class หนึ่งมี
   `friend` มากกว่า 3-4 class โดยไม่มีเหตุผลจากโครงสร้างของโดเมนจริงๆ (เช่น container/
   iterator) มักเป็นสัญญาณว่าการแบ่ง responsibility ระหว่าง class ผิดตั้งแต่ต้น

---

## แบบฝึกหัดท้ายบท

1. เขียน class `Matrix2x2` (เมทริกซ์ 2x2) แล้วเขียน friend function `operator==` ที่
   เปรียบเทียบว่าเมทริกซ์สองตัวมีค่าเท่ากันทุกช่องหรือไม่
2. อธิบายด้วยคำพูดตัวเองว่าทำไม `operator<<` ที่ใช้แค่ public getter (เหมือน `Vector2D`
   ใน Part 51) ถึง**ไม่จำเป็น**ต้องเป็น friend แต่ `operator<<` ของ `Money` ใน Part นี้
   เลือกใช้ friend (คำใบ้: ทั้งสองแบบ compile ผ่านได้ทั้งคู่ ต่างกันที่อะไร)
3. เขียนโค้ดสาธิตว่า friend ไม่ transitive: สร้าง class `A`, `B`, `C` โดยที่ `A` เป็น
   friend ของ `B` และ `B` เป็น friend ของ `C` แล้วพิสูจน์ (ด้วยการลองเขียนโค้ดที่ควร
   compile ไม่ผ่าน แล้วดู error message) ว่า `A` เข้าถึง private ของ `C` ไม่ได้
4. เขียน friend class `LinkedList::ListIterator` เพิ่มเติมจากตัวอย่าง 52.6 ให้มี method
   `reset()` ที่ทำให้ iterator กลับไปเริ่มต้นที่ `head_` ใหม่อีกครั้ง
5. หา class ในโค้ดที่คุณเคยเขียนมาก่อน (หรือในแบบฝึกหัดของ Part 47-51) ที่มี getter/setter
   จำนวนมาก แล้วลองพิจารณาว่ามี method คู่ไหนที่ **ควรรวมเป็น friend function เดียว**
   แทนได้ พร้อมอธิบายเหตุผล
6. อ่านโค้ด Anti-pattern ใน 52.7 อีกครั้ง แล้วเขียนเวอร์ชันที่แก้ไขให้ถูกต้อง (ลบ
   `friend class AuditReport;` ออก และใช้ public interface แทน) ทดสอบว่าผลลัพธ์
   ยังเหมือนเดิม

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>

class Matrix2x2 {
public:
    Matrix2x2(double a, double b, double c, double d) : a_(a), b_(b), c_(c), d_(d) {}

    void print() const {
        std::cout << "[ " << a_ << " " << b_ << " ]\n";
        std::cout << "[ " << c_ << " " << d_ << " ]\n";
    }

    friend bool operator==(const Matrix2x2& lhs, const Matrix2x2& rhs);

private:
    double a_, b_, c_, d_;
};

bool operator==(const Matrix2x2& lhs, const Matrix2x2& rhs) {
    return lhs.a_ == rhs.a_ && lhs.b_ == rhs.b_ &&
           lhs.c_ == rhs.c_ && lhs.d_ == rhs.d_;
}

int main() {
    Matrix2x2 m1(1, 2, 3, 4);
    Matrix2x2 m2(1, 2, 3, 4);
    Matrix2x2 m3(1, 0, 0, 1);

    std::cout << std::boolalpha;
    std::cout << "m1 == m2 ? " << (m1 == m2) << '\n';
    std::cout << "m1 == m3 ? " << (m1 == m3) << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 matrix.cpp -o matrix
./matrix
```

ผลลัพธ์:

```
m1 == m2 ? true
m1 == m3 ? false
```

ในตัวอย่างนี้ `friend bool operator==(...)` จำเป็นจริงๆ เพราะฟังก์ชันต้องเข้าถึง
`a_`, `b_`, `c_`, `d_` ของ **ทั้งสอง** object (`lhs` และ `rhs`) พร้อมกัน ถ้าเขียนเป็น
member function จะเข้าถึง `rhs`'s private members ไม่ได้เว้นแต่จะมี public getter
เพิ่มขึ้นมาสี่ตัว (หนึ่งตัวต่อหนึ่ง field) ซึ่งเปิดเผยรายละเอียดภายในมากเกินความจำเป็น
การใช้ friend ในกรณีนี้จึงสมเหตุสมผล

### แนวทางเฉลยข้อ 4

```cpp
#include <iostream>

class ListIterator;

class LinkedList {
public:
    void push_back(int value) {
        Node* node = new Node{value, nullptr};
        if (!head_) {
            head_ = tail_ = node;
        } else {
            tail_->next = node;
            tail_ = node;
        }
    }

    ~LinkedList() {
        Node* current = head_;
        while (current) {
            Node* next = current->next;
            delete current;
            current = next;
        }
    }

    friend class ListIterator;

private:
    struct Node {
        int value;
        Node* next;
    };

    Node* head_ = nullptr;
    Node* tail_ = nullptr;
};

class ListIterator {
public:
    explicit ListIterator(const LinkedList& list) : list_(&list), current_(list.head_) {}

    bool has_next() const { return current_ != nullptr; }

    int next() {
        int value = current_->value;
        current_ = current_->next;
        return value;
    }

    // method ใหม่ตามโจทย์: กลับไปเริ่มต้นที่ head_ อีกครั้ง
    void reset() {
        current_ = list_->head_;   // เข้าถึง private head_ ได้เพราะ ListIterator เป็น friend
    }

private:
    const LinkedList* list_;
    LinkedList::Node* current_;
};

int main() {
    LinkedList list;
    list.push_back(1);
    list.push_back(2);
    list.push_back(3);

    ListIterator it(list);
    std::cout << "รอบแรก: ";
    while (it.has_next()) {
        std::cout << it.next() << " ";
    }
    std::cout << '\n';

    it.reset();
    std::cout << "หลัง reset: ";
    while (it.has_next()) {
        std::cout << it.next() << " ";
    }
    std::cout << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 list_iterator_reset.cpp -o list_iterator_reset
./list_iterator_reset
```

ผลลัพธ์:

```
รอบแรก: 1 2 3
หลัง reset: 1 2 3
```

`ListIterator` ต้องเก็บ pointer ไปยัง `LinkedList` ต้นทาง (`list_`) เพิ่มเติม เพื่อให้
`reset()` กลับไปอ้างอิง `head_` ล่าสุดได้เสมอ (แทนที่จะเก็บค่า `head_` ตายตัวไว้ตั้งแต่
ตอนสร้าง iterator ซึ่งจะผิดถ้า `head_` เปลี่ยนไปหลังจากนั้น) การเข้าถึง `list_->head_`
ยังคงต้องอาศัยสิทธิ์ friend เช่นเดิม

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจความหมายของ Friend Function และเขียน syntax การประกาศได้ถูกต้อง พร้อมเห็น
  ตัวอย่างจริงที่ `operator<<` จำเป็นต้องเข้าถึง private data ผ่าน friend
- เข้าใจ Friend Class และรู้ว่ามันให้สิทธิ์ **ทุก** member function ของ class นั้นเข้าถึง
  private/protected member ของ class ที่ประกาศไว้
- รู้จักคุณสมบัติพิเศษของ friend สามข้อ: ไม่ symmetric, ไม่ transitive, ไม่ถูกสืบทอด
  พร้อมตัวอย่างโค้ดที่พิสูจน์แต่ละข้อชัดเจน
- วิเคราะห์ข้อถกเถียงเรื่อง "friend ทำลาย Encapsulation หรือไม่" ได้อย่างสมดุล เข้าใจว่า
  Encapsulation ที่แท้จริงคือ "class ควบคุมการเข้าถึงของตัวเอง" ไม่ใช่ "ห้ามเข้าถึงเด็ดขาด"
- เห็นตัวอย่างจริงที่ friend เป็นทางออกที่ดีที่สุด (container กับ iterator ของมันเอง)
  เทียบกับตัวอย่าง Anti-pattern ที่ใช้ friend เกินความจำเป็นทั้งที่มี public interface
  เพียงพออยู่แล้ว
- ได้ checklist สำหรับตัดสินใจว่าเมื่อไหร่ควรใช้ friend ในงานออกแบบจริง

ใน **Part 53** เราจะเรียนเรื่อง **Static Member และการออกแบบระดับ Class** — วิธีสร้าง
ข้อมูลและฟังก์ชันที่เป็นของ "class โดยรวม" ไม่ใช่ของ object แต่ละตัว พร้อมแนะนำ Design
Pattern แรกของหลักสูตรนี้: **Singleton Pattern** เบื้องต้น

**ต่อไป:** [Part 53 — Static Member และ Class Design](./part-053-static-members.md)
