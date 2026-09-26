# Part 47: Encapsulation และ Access Specifier (Step 369–376)

> Module D — เริ่มต้น C++ และ OOP | Part 47 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 369–376
> Part ก่อนหน้า: [Part 46 — Constructor และ Destructor](./part-046-constructors-destructors.md) | Part ถัดไป: [Part 48 — Inheritance](./part-048-inheritance.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความหมายและความแตกต่างของ Access Specifier ทั้งสามตัว: `public`, `private`, `protected`
2. อธิบายได้ว่าทำไมการ "ซ่อนข้อมูล" (Information Hiding) จึงเป็นหัวใจของการออกแบบ Class ที่ดี
   และป้องกันบั๊กประเภท Invalid State ได้อย่างไร
3. เข้าใจความแตกต่างของค่า Default Access ระหว่าง `class` กับ `struct`
4. เขียน Getter/Setter ที่ "มีประโยชน์จริง" แยกแยะได้ว่าเมื่อไหร่ควรมี เมื่อไหร่ไม่ควรมี
   และเมื่อไหร่ควรมีแค่ Getter อย่างเดียว
5. ออกแบบ Public Interface ของ Class ตามหลัก **Tell, Don't Ask**
6. ใช้ `const` Member Function เพื่อสร้างความถูกต้องเชิง const (const-correctness) และเข้าใจ
   แนวคิดเบื้องต้นของ Immutable Object
7. เข้าใจ `mutable` keyword และรู้ว่าควรใช้ในสถานการณ์แบบใด
8. เข้าใจแนวคิดเบื้องต้นของ `protected` เพื่อเตรียมพร้อมสำหรับเรื่อง Inheritance ใน Part 48

---

## 47.1 ทบทวน: ทำไม public field ล้วนถึงเป็นปัญหา (Step 369)

ใน Part 45–46 เราสร้าง Class ที่มี Constructor/Destructor ที่ดีแล้ว แต่ยังไม่ได้พูดถึงประเด็นสำคัญ
อีกข้อหนึ่ง: **ใครควรมีสิทธิ์เข้าถึงและแก้ไขข้อมูลภายใน Object ได้บ้าง?**

ลองดูตัวอย่างที่ยังไม่ได้ทำ Encapsulation:

```cpp
#include <iostream>

// struct ทุกฟิลด์เป็น public โดย default -> ใครก็แก้ไขค่าได้อย่างอิสระ
struct TemperatureBad {
    double celsius;
};

int main() {
    TemperatureBad t{25.0};
    std::cout << "อุณหภูมิเริ่มต้น: " << t.celsius << " C\n";

    // ไม่มีอะไรป้องกันไม่ให้ตั้งค่าที่ผิดตามหลักฟิสิกส์เลย
    // -300 องศาเซลเซียสต่ำกว่าศูนย์สัมบูรณ์ (-273.15 C) ซึ่งเป็นไปไม่ได้ในความจริง
    t.celsius = -300.0;
    std::cout << "อุณหภูมิหลังแก้ไข: " << t.celsius << " C (ค่านี้ผิดตามหลักฟิสิกส์!)\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 temperature_bad.cpp -o temperature_bad
./temperature_bad
```

```
อุณหภูมิเริ่มต้น: 25 C
อุณหภูมิหลังแก้ไข: -300 C (ค่านี้ผิดตามหลักฟิสิกส์!)
```

โปรแกรมนี้ **คอมไพล์ผ่านและรันได้สนิท ไม่มี Warning ใดๆ เลย** แต่ผลลัพธ์กลับผิดตามหลักความจริง
ปัญหาไม่ได้อยู่ที่ Syntax แต่อยู่ที่ **การออกแบบ**: เราเปิดให้ใครก็ได้เข้าถึงและแก้ไข `celsius`
ได้โดยตรง โดยไม่มีจุดใดเลยที่คอยตรวจสอบว่าค่าที่ตั้งเข้ามานั้น "สมเหตุสมผล" หรือไม่

นี่คือปัญหาที่ **Encapsulation (การห่อหุ้ม)** ถูกออกแบบมาเพื่อแก้ไขโดยเฉพาะ แนวคิดหลักคือ:

> **ซ่อนรายละเอียดการเก็บข้อมูลภายในไว้ แล้วเปิดเผยเฉพาะ "พฤติกรรม" (behavior) ที่ผ่านการตรวจสอบ
> แล้วเท่านั้นให้โลกภายนอกใช้งาน**

---

## 47.2 Access Specifier: public, private, protected (Step 370)

C++ มี Access Specifier 3 ระดับ ที่ควบคุมว่าใครเข้าถึง Member (ตัวแปรหรือฟังก์ชันของ Class)
ได้บ้าง:

| Access Specifier | เข้าถึงได้จากภายใน Class เดียวกัน | เข้าถึงได้จาก Class ลูก (Derived Class) | เข้าถึงได้จากภายนอก (เช่น `main`) |
|---|:---:|:---:|:---:|
| `public` | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ❌ |
| `private` | ✅ | ❌ | ❌ |

> `protected` จะยังไม่มีประโยชน์ชัดเจนจนกว่าเราจะรู้จัก **Inheritance** ใน Part 48 — ที่นี่เราจะแค่
> เกริ่นนำให้คุ้นตาไว้ก่อน (ดูหัวข้อ 47.4)

### Syntax การประกาศ

```cpp
class Example {
public:
    // Member ที่นี่เข้าถึงได้จากทุกที่

protected:
    // Member ที่นี่เข้าถึงได้จากภายใน class และ class ลูกเท่านั้น

private:
    // Member ที่นี่เข้าถึงได้จากภายใน class เดียวกันเท่านั้น
};
```

Access Specifier แต่ละคำ **มีผลกับ Member ที่ประกาศตามหลังมันไปเรื่อยๆ จนกว่าจะเจอคำใหม่**
(ไม่ใช่แค่บรรทัดถัดไปบรรทัดเดียว) และสามารถสลับไปมาซ้ำได้หลายครั้งในไฟล์เดียว แม้จะไม่ค่อยนิยม
เพราะทำให้อ่านยาก

### Default Access: class vs struct

จุดที่มือใหม่สับสนบ่อยที่สุดคือความแตกต่างเพียงข้อเดียวระหว่าง `class` กับ `struct` ใน C++:

```cpp
#include <iostream>

// class: สมาชิกเป็น private โดย default
class ClassDemo {
    int x_ = 1;       // private โดยอัตโนมัติ
public:
    int getX() const { return x_; }
};

// struct: สมาชิกเป็น public โดย default
struct StructDemo {
    int y_ = 2;       // public โดยอัตโนมัติ
};

int main() {
    ClassDemo c;
    StructDemo s;

    std::cout << "c.getX() = " << c.getX() << '\n';
    std::cout << "s.y_     = " << s.y_ << '\n';

    // c.x_ = 99;   // ถ้าเปิดบรรทัดนี้จะคอมไพล์ไม่ผ่าน เพราะ x_ เป็น private
    s.y_ = 99;       // บรรทัดนี้คอมไพล์ผ่าน เพราะ y_ เป็น public
    std::cout << "s.y_ หลังแก้ไข = " << s.y_ << '\n';

    return 0;
}
```

```
c.getX() = 1
s.y_     = 2
s.y_ หลังแก้ไข = 99
```

**นี่คือความต่างเพียงข้อเดียวระหว่าง `class` กับ `struct` ใน C++** (ทั้งสองคำสามารถมี Constructor,
Method, Access Specifier, Inheritance ได้เหมือนกันหมด) ตามธรรมเนียมของวงการ:

- ใช้ `struct` สำหรับกลุ่มข้อมูลง่ายๆ ที่ไม่มี Invariant ต้องรักษา (Plain Old Data)
  เช่น `struct Point { double x, y; };`
- ใช้ `class` เมื่อ Object มีพฤติกรรมซับซ้อน มี Invariant ที่ต้องป้องกัน หรือต้องซ่อนรายละเอียด
  การ implement ไว้ — ซึ่งเป็นกรณีส่วนใหญ่ที่เราจะเจอตลอดหลักสูตรนี้

### ลองเข้าถึง private จากภายนอก

```cpp
#include <iostream>

class Temperature {
public:
    explicit Temperature(double celsius) : celsius_(celsius) {}
    double celsius() const { return celsius_; }

private:
    double celsius_;
};

int main() {
    Temperature t(25.0);
    std::cout << t.celsius_ << '\n';   // ผิด: celsius_ เป็น private เข้าถึงจากภายนอกไม่ได้
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 bad_access.cpp -o bad_access
```

```
bad_access.cpp: In function 'int main()':
bad_access.cpp:14:20: error: 'double Temperature::celsius_' is private within this context
   14 |     std::cout << t.celsius_ << '\n';
      |                    ^~~~~~~~
bad_access.cpp:9:12: note: declared private here
    9 |     double celsius_;
      |            ^~~~~~~~
bad_access.cpp:14:20: note: field 'double Temperature::celsius_' can be accessed via 'double Temperature::celsius() const'
```

สังเกตว่า Compiler ฉลาดพอที่จะแนะนำเราด้วยว่า "ใช้ `celsius()` แทนสิ" — นี่คือประโยชน์อย่างหนึ่ง
ของการเปิด Warning ครบ: มันช่วยสอนเราแบบ real-time เลยว่าควรแก้โค้ดอย่างไร

---

## 47.3 ทำไมต้อง Encapsulate: Information Hiding และการป้องกัน Invalid State (Step 371)

Encapsulation มีเหตุผลหลัก 2 ข้อ:

### เหตุผลที่ 1: ป้องกัน Invalid State (Invariant Protection)

**Invariant** คือ "กฎที่ต้องเป็นจริงเสมอ" ตลอดอายุของ Object เช่น อุณหภูมิเซลเซียสต้องไม่ต่ำกว่า
-273.15 หรือยอดเงินในบัญชีธนาคารต้องไม่ติดลบ ถ้าเราให้ข้อมูลเป็น `public` ตรงๆ ไม่มีทางบังคับ
Invariant เหล่านี้ได้เลย เพราะใครก็แก้ไขค่าได้โดยตรงโดยไม่ผ่านการตรวจสอบใดๆ

มาแก้ตัวอย่างอุณหภูมิด้วย Encapsulation ที่ถูกต้อง:

```cpp
#include <iostream>
#include <stdexcept>

class Temperature {
public:
    explicit Temperature(double celsius) { setCelsius(celsius); }

    double celsius() const { return celsius_; }

    void setCelsius(double celsius) {
        if (celsius < kAbsoluteZeroC) {
            throw std::invalid_argument(
                "อุณหภูมิต่ำกว่าศูนย์สัมบูรณ์ (-273.15 C) เป็นไปไม่ได้จริง");
        }
        celsius_ = celsius;
    }

    static constexpr double kAbsoluteZeroC = -273.15;

private:
    double celsius_ = 0.0;
};

int main() {
    Temperature t(25.0);
    std::cout << "อุณหภูมิ: " << t.celsius() << " C\n";

    try {
        t.setCelsius(-300.0);   // ผิดหลักฟิสิกส์ -> ต้อง reject
    } catch (const std::invalid_argument& e) {
        std::cout << "ตั้งค่าไม่สำเร็จ: " << e.what() << '\n';
    }

    std::cout << "อุณหภูมิยังคงเดิม: " << t.celsius() << " C\n";
    return 0;
}
```

```
อุณหภูมิ: 25 C
ตั้งค่าไม่สำเร็จ: อุณหภูมิต่ำกว่าศูนย์สัมบูรณ์ (-273.15 C) เป็นไปไม่ได้จริง
อุณหภูมิยังคงเดิม: 25 C
```

ตอนนี้ **ไม่มีทางใดเลย** ที่จะสร้าง `Temperature` Object ที่มีค่าผิดหลักฟิสิกส์ได้ ไม่ว่าจะพยายามผ่าน
Constructor หรือผ่าน `setCelsius()` — เพราะทางเข้าเดียวที่แก้ไข `celsius_` ได้คือผ่าน
`setCelsius()` ที่ตรวจสอบเงื่อนไขไว้แล้ว (`std::invalid_argument` และ `try/catch` จะเรียนเจาะลึก
ใน Part 54 เรื่อง Exception Handling — ตอนนี้ขอให้เข้าใจแค่ว่ามันคือกลไก "โยน error" เมื่อค่าที่
ได้รับผิดเงื่อนไข)

อีกตัวอย่างที่คลาสสิกมาก คือบัญชีธนาคาร ที่ยอดเงินต้องไม่ติดลบ และการฝาก/ถอนต้องเป็นจำนวนบวก:

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

class BankAccount {
public:
    BankAccount(std::string owner, double initial_balance)
        : owner_(std::move(owner)), balance_(0.0) {
        if (initial_balance < 0.0) {
            throw std::invalid_argument("ยอดเงินเริ่มต้นต้องไม่ติดลบ");
        }
        balance_ = initial_balance;
    }

    void deposit(double amount) {
        if (amount <= 0.0) {
            throw std::invalid_argument("ฝากเงินต้องเป็นจำนวนบวกเท่านั้น");
        }
        balance_ += amount;
    }

    void withdraw(double amount) {
        if (amount <= 0.0) {
            throw std::invalid_argument("ถอนเงินต้องเป็นจำนวนบวกเท่านั้น");
        }
        if (amount > balance_) {
            throw std::invalid_argument("ยอดเงินไม่พอสำหรับการถอน");
        }
        balance_ -= amount;
    }

    double balance() const { return balance_; }
    const std::string& owner() const { return owner_; }

private:
    std::string owner_;
    double balance_;
};

int main() {
    BankAccount acc("Somchai", 1000.0);

    acc.deposit(500.0);
    std::cout << acc.owner() << " มียอดเงิน " << acc.balance() << " บาท\n";

    try {
        acc.withdraw(100000.0);   // เกินยอดเงินที่มี
    } catch (const std::invalid_argument& e) {
        std::cout << "ถอนเงินไม่สำเร็จ: " << e.what() << '\n';
    }

    std::cout << "ยอดเงินคงเหลือ (ไม่เปลี่ยนแปลง): " << acc.balance() << " บาท\n";
    return 0;
}
```

```
Somchai มียอดเงิน 1500 บาท
ถอนเงินไม่สำเร็จ: ยอดเงินไม่พอสำหรับการถอน
ยอดเงินคงเหลือ (ไม่เปลี่ยนแปลง): 1500 บาท
```

สังเกตว่า `balance_` เป็น `private` ทั้งหมด และไม่มี `setBalance()` เลย — ทางเดียวที่ยอดเงินจะ
เปลี่ยนได้คือผ่าน `deposit()` หรือ `withdraw()` เท่านั้น ซึ่งทั้งคู่ตรวจสอบเงื่อนไขไว้แล้ว การไม่มี
`setBalance()` **เป็นการตัดสินใจออกแบบที่ตั้งใจ** ไม่ใช่ความสะเพร่า

### เหตุผลที่ 2: Information Hiding (ซ่อนรายละเอียดการ Implement)

Encapsulation ยังทำให้เราสามารถ **เปลี่ยนวิธีเก็บข้อมูลภายในได้ โดยไม่กระทบโค้ดภายนอกที่ใช้งาน
Class นี้เลย** ตราบใดที่ Public Interface (ฟังก์ชัน `public` ทั้งหมด) ยังทำงานเหมือนเดิม

ตัวอย่างเช่น ถ้าวันหนึ่งเราตัดสินใจเปลี่ยน `BankAccount` ให้เก็บ `balance_` เป็นหน่วยสตางค์
(จำนวนเต็ม, ไม่ใช้ `double` เพื่อหลีกเลี่ยงปัญหาความคลาดเคลื่อนของ floating-point) โค้ดที่เรียกใช้
`acc.balance()`, `acc.deposit()`, `acc.withdraw()` จากภายนอก **ไม่ต้องแก้ไขอะไรเลยแม้แต่บรรทัด
เดียว** เพราะมันไม่เคยรู้และไม่จำเป็นต้องรู้เลยว่าข้างในเก็บข้อมูลด้วยชนิดอะไร — นี่คือพลังของการ
"ซ่อนรายละเอียด" ที่ทำให้โค้ดขนาดใหญ่ดูแลรักษาง่ายขึ้นมหาศาล

---

## 47.4 protected: เกริ่นนำก่อนเรื่อง Inheritance (Step 372)

`protected` มีพฤติกรรมเหมือน `private` ทุกประการ **ยกเว้นข้อเดียว**: Class ที่สืบทอด (Derived
Class) จาก Class นี้ สามารถเข้าถึง Member ที่เป็น `protected` ได้โดยตรง ในขณะที่ยังคงปิดกั้น
การเข้าถึงจากภายนอก (เช่นจาก `main`) เหมือน `private`

```cpp
#include <iostream>
#include <string>

// ตัวอย่างเบื้องต้นของ protected — จะเข้าใจเต็มรูปแบบใน Part 48 (Inheritance)
class Animal {
protected:
    // protected: เข้าถึงได้จากภายใน class เดียวกัน "และจาก class ลูกที่สืบทอด"
    // แต่เข้าถึงจากภายนอก (เช่นใน main) ไม่ได้ เหมือน private
    std::string name_;

public:
    explicit Animal(std::string name) : name_(std::move(name)) {}
};

class Dog : public Animal {
public:
    explicit Dog(std::string name) : Animal(std::move(name)) {}

    void bark() const {
        // เข้าถึง name_ ได้ เพราะ Dog สืบทอดมาจาก Animal และ name_ เป็น protected
        std::cout << name_ << ": โฮ่ง โฮ่ง!\n";
    }
};

int main() {
    Dog d("โปงลาง");
    d.bark();

    // d.name_ = "ใหม่";  // ผิด: name_ เป็น protected เข้าถึงจากภายนอก class ไม่ได้
    return 0;
}
```

```
โปงลาง: โฮ่ง โฮ่ง!
```

ตอนนี้ให้จำแค่หลักการนี้ไว้ก่อน: **`protected` = private สำหรับโลกภายนอก แต่เปิดให้ class ลูก
เข้าถึงได้** เราจะกลับมาเจาะลึกเรื่องนี้อย่างละเอียดใน Part 48 พร้อมพูดถึงข้อควรระวังว่าทำไม
นักออกแบบมืออาชีพจำนวนมากถึงระมัดระวังการใช้ `protected` มากกว่าที่คิด (เพราะมันคือการเปิด
"รอยรั่ว" ของ Encapsulation ให้ Class ลูกในระดับหนึ่ง)

---

## 47.5 Getter/Setter ที่ดี ไม่ใช่แค่ห่อ public field ทุกตัว (Step 373)

ข้อผิดพลาดที่พบบ่อยที่สุดของมือใหม่ (และแม้แต่โปรแกรมเมอร์ที่มีประสบการณ์บางคน) คือคิดว่า
"Encapsulation = ทำทุก field ให้เป็น private แล้วสร้าง getter/setter คู่กันให้ครบทุกตัว"
ซึ่ง **ไม่ใช่ Encapsulation ที่แท้จริงเลย** มันเป็นแค่ public field ที่ห่อด้วยฟังก์ชันเพิ่มขึ้นมาเฉยๆ
โดยไม่มีการป้องกันอะไรเพิ่มขึ้นจริง

### ตัวอย่างที่ "ดูเหมือนดี" แต่จริงๆ แล้วแย่ (Anemic Getter/Setter)

```cpp
#include <iostream>

// ตัวอย่างการออกแบบที่ "ไม่ดี" — ห่อทุก field ด้วย getter/setter แบบไม่คิดอะไร
// (Anemic class / getter-setter ที่ไม่มีประโยชน์ใดๆ เพิ่มจาก public field ธรรมดา)
class RectangleAnemic {
public:
    void setWidth(double w) { width_ = w; }
    double getWidth() const { return width_; }

    void setHeight(double h) { height_ = h; }
    double getHeight() const { return height_; }

    void setArea(double a) { area_ = a; }     // อันตราย: ให้คนนอกตั้งค่า area เองได้ตรงๆ
    double getArea() const { return area_; }

private:
    double width_ = 0.0;
    double height_ = 0.0;
    double area_ = 0.0;
};

int main() {
    RectangleAnemic r;
    r.setWidth(4.0);
    r.setHeight(5.0);
    r.setArea(999.0);   // ตั้งค่า area มั่วๆ ที่ไม่สัมพันธ์กับ width*height เลย!

    std::cout << "width=" << r.getWidth() << " height=" << r.getHeight()
              << " area=" << r.getArea() << " (ผิด! ควรเป็น 20)\n";
    return 0;
}
```

```
width=4 height=5 area=999 (ผิด! ควรเป็น 20)
```

ปัญหาชัดเจน: `area_` ควรจะเท่ากับ `width_ * height_` เสมอ แต่ `setArea()` เปิดช่องให้คนนอก
ตั้งค่าที่ไม่สัมพันธ์กันได้เลย — Class นี้ **ไม่ได้ป้องกัน Invalid State อะไรเลย** แม้จะมี `private`
ครบทุก field ก็ตาม เพราะ setter แค่ก็อปปี้ค่าเข้าไปตรงๆ โดยไม่คิดอะไร

### ออกแบบใหม่: area เป็น "พฤติกรรม" ไม่ใช่ "state ที่เก็บแยก"

```cpp
#include <iostream>
#include <stdexcept>

// ตัวอย่างการออกแบบที่ "ดี" — area ไม่ใช่ state ที่เก็บไว้ แต่เป็น "พฤติกรรม" ที่คำนวณสด
// ไม่มี setArea() เลย เพราะ area ไม่ใช่สิ่งที่ผู้ใช้ควรตั้งค่าได้โดยตรง
class Rectangle {
public:
    Rectangle(double width, double height) { setWidth(width); setHeight(height); }

    void setWidth(double w) {
        if (w <= 0.0) throw std::invalid_argument("width ต้องเป็นบวก");
        width_ = w;
    }
    void setHeight(double h) {
        if (h <= 0.0) throw std::invalid_argument("height ต้องเป็นบวก");
        height_ = h;
    }

    double width() const { return width_; }
    double height() const { return height_; }

    // area เป็น method ที่คำนวณจาก width/height เสมอ -> ไม่มีทางไม่ตรงกัน (invalid state)
    double area() const { return width_ * height_; }
    double perimeter() const { return 2.0 * (width_ + height_); }

private:
    double width_ = 0.0;
    double height_ = 0.0;
};

int main() {
    Rectangle r(4.0, 5.0);
    std::cout << "width=" << r.width() << " height=" << r.height()
              << " area=" << r.area() << " perimeter=" << r.perimeter() << '\n';

    r.setWidth(10.0);
    std::cout << "หลังเปลี่ยน width -> area=" << r.area() << " (คำนวณใหม่อัตโนมัติ)\n";
    return 0;
}
```

```
width=4 height=5 area=20 perimeter=18
หลังเปลี่ยน width -> area=50 (คำนวณใหม่อัตโนมัติ)
```

หลักการสำคัญ: **ถ้าค่าใดค่าหนึ่งสามารถคำนวณได้จากค่าอื่นเสมอ อย่าเก็บมันเป็น field แยก**
เพราะจะเปิดช่องให้ค่าสองตัวไม่ตรงกัน (out of sync) ได้ ให้คำนวณมันเป็น method แทน

### บางฟิลด์ไม่ควรมี setter เลย (Read-Only Property)

บาง field เป็นค่าที่ต้องกำหนดครั้งเดียวตอนสร้าง Object แล้วห้ามเปลี่ยนตลอดอายุของ Object นั้น
เช่น รหัสพนักงาน (Employee ID) วิธีที่ถูกต้องคือให้มีเฉพาะ Getter โดยไม่มี Setter คู่กันเลย และ
ประกาศ field นั้นเป็น `const` เพื่อให้ Compiler ช่วยการันตีอีกชั้นหนึ่งด้วย:

```cpp
#include <iostream>
#include <string>

// ตัวอย่าง getter ที่ไม่มี setter คู่กันเลย (read-only property)
// id_ ถูกกำหนดครั้งเดียวตอนสร้าง object แล้วห้ามเปลี่ยนตลอดอายุของ object
class Employee {
public:
    Employee(int id, std::string name) : id_(id), name_(std::move(name)) {}

    int id() const { return id_; }              // มี getter อย่างเดียว ไม่มี setId()
    const std::string& name() const { return name_; }

    void setName(std::string name) { name_ = std::move(name); }  // ชื่อเปลี่ยนได้ (ไม่ใช่ invariant)

private:
    const int id_;        // const member: กำหนดได้ครั้งเดียวใน constructor เท่านั้น
    std::string name_;
};

int main() {
    Employee e(1001, "Somchai");
    std::cout << "id=" << e.id() << " name=" << e.name() << '\n';

    e.setName("Somsri");
    std::cout << "หลังเปลี่ยนชื่อ: id=" << e.id() << " name=" << e.name() << '\n';

    // e.id_ = 2002; // ผิดสองต่อ: ทั้ง private และเป็น const member ด้วย
    return 0;
}
```

```
id=1001 name=Somchai
หลังเปลี่ยนชื่อ: id=1001 name=Somsri
```

### เช็คลิสต์ก่อนเขียน Getter/Setter

ก่อนจะเพิ่ม Getter หรือ Setter ให้ field ไหน ให้ถามตัวเองทุกครั้ง:

| คำถาม | ถ้าคำตอบคือ... |
|---|---|
| ค่านี้คำนวณได้จาก field อื่นหรือไม่? | อย่าเก็บเป็น field แยก ให้ทำเป็น method ที่คำนวณสด |
| ค่านี้ต้องคงที่ตลอดอายุ Object หรือไม่? | มีแค่ Getter ประกาศ field เป็น `const` |
| การตั้งค่าใหม่มีเงื่อนไขที่ต้องตรวจสอบหรือไม่? | Setter ต้อง validate เสมอ (throw หรือ reject ถ้าไม่ผ่าน) |
| ผู้ใช้ Class จำเป็นต้องรู้ค่านี้จริงหรือไม่? | ถ้าไม่จำเป็น อย่าเปิด Getter เลย เก็บเป็นรายละเอียดภายในไป |

---

## 47.6 การออกแบบ Public Interface ที่ดี: Tell, Don't Ask (Step 374)

หลักการออกแบบที่ช่วยตัดสินใจว่า Class ควรมี Method อะไรบ้างคือ **Tell, Don't Ask**:

> อย่า "ถาม" ข้อมูลภายในของ Object ออกมาแล้วเอาไปประมวลผลเองข้างนอก
> ให้ "บอก" Object ว่าต้องการให้มันทำอะไร แล้วปล่อยให้มันจัดการ logic ภายในของตัวเอง

### แบบ "Ask" (ไม่ดี)

```cpp
#include <iostream>
#include <vector>

// แบบ "Ask" (ไม่ดี): ให้โค้ดภายนอกดึงข้อมูลภายในออกไปคำนวณเอง
// ทำให้ logic การคำนวณราคารวมกระจัดกระจายอยู่นอก class -> แก้ไข/ทดสอบยาก และผิดพลาดง่าย
class ShoppingCartAsk {
public:
    void addItem(double price) { prices_.push_back(price); }
    const std::vector<double>& prices() const { return prices_; }  // เปิดโครงสร้างภายในออกไปหมด

private:
    std::vector<double> prices_;
};

int main() {
    ShoppingCartAsk cart;
    cart.addItem(100.0);
    cart.addItem(250.0);

    // โค้ดภายนอกต้อง "ถาม" (ask) ข้อมูลออกมาแล้วคำนวณเอง
    double total = 0.0;
    for (double p : cart.prices()) total += p;
    std::cout << "ยอดรวม (คำนวณนอก class): " << total << " บาท\n";
    return 0;
}
```

ปัญหาของแบบนี้คือ ถ้ามีจุดคำนวณยอดรวมแบบนี้กระจายอยู่ 10 ที่ในโปรแกรม แล้ววันหนึ่งกฎการคำนวณ
เปลี่ยน (เช่น ต้องหักส่วนลดก่อนรวม) เราต้องไปตามแก้ไขทั้ง 10 ที่ และเสี่ยงพลาดตกหล่นบางจุด

### แบบ "Tell" (ดี)

```cpp
#include <iostream>
#include <vector>

// แบบ "Tell" (ดี): บอกให้ object ทำงานให้ (tell) แทนที่จะถามข้อมูลออกมาคำนวณเอง
// logic การคำนวณอยู่ในที่เดียว (class เดียว) แก้ไข/ทดสอบง่าย และปลอดภัยกว่า
class ShoppingCart {
public:
    void addItem(double price) { prices_.push_back(price); }

    double total() const {
        double sum = 0.0;
        for (double p : prices_) sum += p;
        return sum;
    }

    int itemCount() const { return static_cast<int>(prices_.size()); }

private:
    std::vector<double> prices_;   // ไม่มี getter เปิดโครงสร้างภายในออกไปเลย
};

int main() {
    ShoppingCart cart;
    cart.addItem(100.0);
    cart.addItem(250.0);

    // "บอก" ให้ cart คำนวณยอดรวมให้เอง ไม่ต้องรู้เลยว่าข้างในเก็บข้อมูลด้วยโครงสร้างอะไร
    std::cout << "ยอดรวม: " << cart.total() << " บาท จาก " << cart.itemCount() << " ชิ้น\n";
    return 0;
}
```

```
ยอดรวม: 350 บาท จาก 2 ชิ้น
```

ตอนนี้ logic การคำนวณยอดรวมอยู่ **ที่เดียว** คือใน `total()` ถ้ากฎเปลี่ยน เราแก้ไขแค่จุดเดียว
และโค้ดที่เรียกใช้ `ShoppingCart` จากภายนอกก็ไม่จำเป็นต้องรู้เลยว่าข้างในเก็บราคาด้วย
`std::vector<double>` หรือโครงสร้างข้อมูลอื่น — นี่คือ Information Hiding ที่ทำงานร่วมกับหลัก
Tell, Don't Ask ได้อย่างสมบูรณ์

**คำแนะนำในการออกแบบ Public Interface:**

1. เปิดเผยเฉพาะ Method ที่ตอบคำถาม "Class นี้ทำอะไรให้ได้บ้าง (What)" ไม่ใช่ "ข้างในเก็บข้อมูล
   อย่างไร (How)"
2. Interface ควรมีขนาดเล็กที่สุดเท่าที่จำเป็น (Minimal Interface) — ยิ่งเปิดเผย Method น้อย ยิ่งมี
   อิสระในการเปลี่ยนแปลง Implementation ภายในในอนาคตมากขึ้น
3. ถ้าพบว่าตัวเองต้องเขียน Loop วนอ่านข้อมูลภายในของ Object อื่นซ้ำๆ ในหลายที่ของโปรแกรม
   นั่นเป็นสัญญาณว่าควรย้าย Logic นั้นเข้าไปเป็น Method ของ Class นั้นแทน

---

## 47.7 const Member Function และ Immutable Object เบื้องต้น (Step 375)

เราเห็น `const` ต่อท้ายฟังก์ชันสมาชิกมาตลอดตั้งแต่ Part 45 (เช่น `double celsius() const`)
ตอนนี้ถึงเวลาทำความเข้าใจมันอย่างละเอียด

### const Member Function คืออะไร

`const` ที่ต่อท้ายฟังก์ชันสมาชิก คือคำสัญญาต่อ Compiler ว่า **"ฟังก์ชันนี้จะไม่แก้ไข state ใดๆ
ของ Object เลย"** Compiler จะบังคับใช้สัญญานี้อย่างเข้มงวด: ถ้าพยายามแก้ไข Member Variable
ภายในฟังก์ชันที่ประกาศเป็น `const` จะ Compile Error ทันที

```cpp
#include <iostream>
#include <string>

class Point2D {
public:
    Point2D(double x, double y) : x_(x), y_(y) {}

    double x() const { return x_; }   // const member function: สัญญาว่าจะไม่แก้ไข object
    double y() const { return y_; }

    void moveBy(double dx, double dy) {   // ไม่ใช่ const เพราะแก้ไข state จริง
        x_ += dx;
        y_ += dy;
    }

    void print() const {   // const เพราะแค่พิมพ์ค่า ไม่แก้ไขอะไร
        std::cout << "(" << x_ << ", " << y_ << ")\n";
    }

private:
    double x_;
    double y_;
};

void printPoint(const Point2D& p) {   // รับเป็น const reference -> การันตีว่าฟังก์ชันนี้ไม่แก้ไข p
    // p.moveBy(1, 1);   // ผิด: เรียก non-const method จาก const object ไม่ได้
    p.print();           // ถูก: print() เป็น const method เรียกได้
}

int main() {
    Point2D p(1.0, 2.0);
    p.print();
    p.moveBy(3.0, 4.0);
    p.print();

    const Point2D fixedPoint(10.0, 20.0);
    printPoint(fixedPoint);   // fixedPoint เป็น const -> เรียกได้เฉพาะ const method เท่านั้น

    return 0;
}
```

```
(1, 2)
(4, 6)
(10, 20)
```

### กฎสำคัญ: const object เรียกได้เฉพาะ const method

```cpp
#include <iostream>

class Counter {
public:
    explicit Counter(int start) : value_(start) {}

    int value() const { return value_; }

    void increment() { ++value_; }   // ไม่ใช่ const เพราะแก้ไข value_

private:
    int value_;
};

int main() {
    const Counter c(0);
    c.increment();   // ผิด: increment() ไม่ใช่ const method เรียกจาก const object ไม่ได้
    std::cout << c.value() << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 const_violation.cpp -o const_violation
```

```
const_violation.cpp: In function 'int main()':
const_violation.cpp:17:16: error: passing 'const Counter' as 'this' argument discards qualifiers [-fpermissive]
   17 |     c.increment();
      |     ~~~~~~~~~~~^~
const_violation.cpp:9:10: note:   in call to 'void Counter::increment()'
```

**นี่คือประโยชน์อันมหาศาลของ const-correctness**: ถ้าเราออกแบบ Class ให้มี `const` ครบถ้วน
ถูกต้อง Compiler จะช่วยจับบั๊กประเภท "พยายามแก้ไขค่าที่ไม่ควรแก้ไข" ให้ตั้งแต่ตอน Compile
โดยที่เราไม่ต้องรันโปรแกรมเลยด้วยซ้ำ

### แนวคิด Immutable Object เบื้องต้น

**Immutable Object** คือ Object ที่ **ไม่มีทางแก้ไขค่าได้เลยหลังจากสร้างเสร็จแล้ว** วิธีสร้าง
Immutable Object แบบง่ายที่สุดใน C++ คือทำให้ Member Variable ทั้งหมดเป็น `const` และ Class
มีแต่ Constructor กับ Method ที่เป็น `const` เท่านั้น ไม่มี Method ที่แก้ไข state เลย:

```cpp
class ImmutablePoint {
public:
    ImmutablePoint(double x, double y) : x_(x), y_(y) {}
    double x() const { return x_; }
    double y() const { return y_; }
    // ไม่มี setX(), setY(), หรือ method ที่แก้ไข state ใดๆ เลย

private:
    const double x_;
    const double y_;
};
```

Object ประเภทนี้มีข้อดีสำคัญมากในโปรแกรมที่ซับซ้อน โดยเฉพาะเมื่อมีหลาย Thread ทำงานพร้อมกัน
(จะเรียนเรื่อง Thread ใน Module I): **Object ที่ไม่มีทางเปลี่ยนแปลงค่าได้เลย ไม่มีทาง**เกิดปัญหา
Race Condition จากการแก้ไขพร้อมกัน เพราะไม่มีใครแก้ไขมันได้ตั้งแต่แรก

### mutable: ข้อยกเว้นที่จำเป็นสำหรับ Cache

บางครั้งเรามี field ที่เป็นแค่ "รายละเอียดภายใน" (implementation detail) ไม่ใช่ "state เชิงตรรกะ"
ของ Object เช่นค่าที่คำนวณไว้ล่วงหน้าเก็บเป็น Cache — ในกรณีนี้ C++ มี keyword `mutable` ที่
อนุญาตให้แก้ไข field นั้นได้แม้อยู่ใน `const` Method:

```cpp
#include <iostream>
#include <cmath>

// ตัวอย่าง mutable: cache ผลลัพธ์ที่คำนวณหนักๆ ไว้ภายใน const method ได้
// mutable บอกคอมไพเลอร์ว่า field นี้ "แก้ไขได้แม้ใน const method" เพราะมันเป็นแค่ cache
// ไม่ใช่ state ทางตรรกะ (logical state) ของ object จริงๆ
class Circle {
public:
    explicit Circle(double radius) : radius_(radius) {}

    double radius() const { return radius_; }

    double area() const {
        if (!area_cached_) {
            std::cout << "  (คำนวณ area ใหม่ ครั้งแรกเท่านั้น)\n";
            cached_area_ = M_PI * radius_ * radius_;
            area_cached_ = true;
        }
        return cached_area_;
    }

private:
    double radius_;
    mutable double cached_area_ = 0.0;
    mutable bool area_cached_ = false;
};

int main() {
    const Circle c(5.0);   // c เป็น const object แต่ area() ยังแก้ cache ภายในได้เพราะ mutable

    double first = c.area();
    std::cout << "area ครั้งที่ 1 = " << first << '\n';

    double second = c.area();
    std::cout << "area ครั้งที่ 2 = " << second << " (ใช้ค่าที่ cache ไว้ ไม่คำนวณซ้ำ)\n";
    return 0;
}
```

```
  (คำนวณ area ใหม่ ครั้งแรกเท่านั้น)
area ครั้งที่ 1 = 78.5398
area ครั้งที่ 2 = 78.5398 (ใช้ค่าที่ cache ไว้ ไม่คำนวณซ้ำ)
```

> **ข้อควรระวัง**: `mutable` ควรใช้เฉพาะกับ field ที่เป็น "รายละเอียดภายใน" จริงๆ เช่น cache,
> mutex สำหรับ lock ภายใน หรือ debug counter เท่านั้น **ห้าม**ใช้ `mutable` เพื่อ "แอบ" แก้ไข
> state เชิงตรรกะของ Object จาก const method เพราะจะทำลายความหมายของ const-correctness
> ทั้งหมดที่เราพยายามสร้างไว้

---

## 47.8 ตัวอย่างสรุปรวม: ออกแบบ Class ให้ Encapsulate อย่างสมบูรณ์ (Step 376)

มารวมทุกหลักการที่เรียนมาใน Part นี้ไว้ในตัวอย่างเดียว: Class `StudentGrade` ที่:

- private data ทั้งหมด — ไม่มี public field เลย
- Invariant: คะแนนต้องอยู่ในช่วง 0–100 เสมอ (บังคับผ่าน setter ที่ validate)
- `id` เป็น read-only (const member ไม่มี setter คู่กัน)
- `grade()` เป็น "พฤติกรรม" ที่คำนวณจากคะแนนเสมอ ไม่ใช่ field ที่เก็บแยกต่างหาก
- const member function ครบทุกตัวที่ไม่แก้ไข state

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

/*
 * StudentGrade — ตัวอย่างสรุปรวมของ Encapsulation ที่ดี
 * - private data ทั้งหมด
 * - invariant: score ต้องอยู่ในช่วง 0-100 เสมอ
 * - id เป็น read-only (const member, ไม่มี setter)
 * - grade() เป็น "พฤติกรรม" ที่คำนวณจาก score เสมอ ไม่ใช่ state ที่เก็บแยก
 * - const member function ครบทุกตัวที่ไม่แก้ไข state
 */
class StudentGrade {
public:
    StudentGrade(int id, std::string name, double score)
        : id_(id), name_(std::move(name)), score_(0.0) {
        setScore(score);
    }

    int id() const { return id_; }
    const std::string& name() const { return name_; }
    double score() const { return score_; }

    void setScore(double score) {
        if (score < 0.0 || score > 100.0) {
            throw std::invalid_argument("คะแนนต้องอยู่ระหว่าง 0 ถึง 100 เท่านั้น");
        }
        score_ = score;
    }

    // grade ไม่ใช่ field ที่เก็บแยก แต่คำนวณจาก score_ เสมอ -> ไม่มีทางไม่ตรงกัน
    char grade() const {
        if (score_ >= 80.0) return 'A';
        if (score_ >= 70.0) return 'B';
        if (score_ >= 60.0) return 'C';
        if (score_ >= 50.0) return 'D';
        return 'F';
    }

    void print() const {
        std::cout << "[" << id_ << "] " << name_
                  << " คะแนน=" << score_ << " เกรด=" << grade() << '\n';
    }

private:
    const int id_;
    std::string name_;
    double score_;
};

int main() {
    StudentGrade s1(1, "Somchai", 85.5);
    StudentGrade s2(2, "Somsri", 42.0);

    s1.print();
    s2.print();

    s2.setScore(65.0);   // แก้ไขคะแนนผ่าน setter ที่ validate แล้ว
    std::cout << "หลังแก้ไขคะแนน:\n";
    s2.print();

    try {
        s1.setScore(150.0);   // เกินขอบเขต -> ต้องถูกปฏิเสธ
    } catch (const std::invalid_argument& e) {
        std::cout << "แก้ไขคะแนนไม่สำเร็จ: " << e.what() << '\n';
    }

    return 0;
}
```

```
[1] Somchai คะแนน=85.5 เกรด=A
[2] Somsri คะแนน=42 เกรด=F
หลังแก้ไขคะแนน:
[2] Somsri คะแนน=65 เกรด=C
แก้ไขคะแนนไม่สำเร็จ: คะแนนต้องอยู่ระหว่าง 0 ถึง 100 เท่านั้น
```

Class นี้แสดงให้เห็นว่า **Encapsulation ที่ดีไม่ใช่แค่การใส่ `private` ให้ทุก field** แต่เป็นการ
**ออกแบบ Public Interface อย่างตั้งใจ** ให้เป็นไปไม่ได้เลยที่จะสร้าง Object ในสถานะที่ผิดพลาด
ไม่ว่าผู้ใช้ Class จะพยายามเรียกใช้ Method ในลำดับใดก็ตาม

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ห่อทุก field ด้วย getter/setter โดยไม่คิด (Anemic Getter/Setter)** — เหมือนตัวอย่าง
   `RectangleAnemic` ในหัวข้อ 47.5 การมี `private` field แต่ทำ getter/setter ที่ไม่ validate
   อะไรเลยไม่ต่างอะไรจากการมี public field เพียงแค่พิมพ์โค้ดยาวขึ้น

2. **ลืมใส่ `const` ท้าย Getter** — ถ้าเขียน `double celsius() { return celsius_; }` โดยไม่มี
   `const` ฟังก์ชันนี้จะเรียกจาก `const` Object หรือ `const Temperature&` parameter ไม่ได้เลย
   ทำให้เขียนโค้ดที่รับ `const&` เป็น parameter ไม่ได้อย่างที่ควรจะเป็น เป็นปัญหาที่พบบ่อยมาก
   และมักตามแก้ยากเมื่อโค้ดใหญ่ขึ้น — **กฎทอง: Method ใดไม่แก้ไข state ต้องใส่ `const` เสมอ**

3. **เก็บค่าที่คำนวณได้จากค่าอื่นเป็น field แยก** — เช่นเก็บ `area_` แยกจาก `width_`/`height_`
   ทำให้เกิดความเสี่ยงที่ค่าทั้งสองจะไม่ตรงกัน (Out of Sync) ถ้ามี Setter ให้แก้ไข `width_`
   แต่ลืมอัปเดต `area_` ด้วย

4. **ใช้ `mutable` เพื่อ "แอบ" แก้ไข state เชิงตรรกะ** — เช่นใช้ `mutable` กับ field ที่มีผลต่อ
   พฤติกรรมที่สังเกตได้จากภายนอก (observable behavior) ซึ่งขัดกับเจตนารมณ์ของ const-correctness
   `mutable` ควรใช้กับ Cache หรือรายละเอียดภายในที่ไม่กระทบผลลัพธ์ที่มองเห็นได้เท่านั้น

5. **เปิดเผย Reference หรือ Pointer ไปยัง Internal Data โดยไม่จำเป็น** — เช่นในตัวอย่าง
   `ShoppingCartAsk` ที่คืน `const std::vector<double>&` ตรงๆ แม้จะเป็น `const` ก็ยังเปิดเผย
   ว่า "ข้างในเก็บด้วย vector" ซึ่งผูกมัด Implementation ไว้ ทำให้เปลี่ยนโครงสร้างข้อมูลภายในทีหลัง
   ยากขึ้น ควรเปิดเผยเฉพาะ Method ระดับพฤติกรรม (เช่น `total()`, `itemCount()`) แทน

6. **ลืมว่า `struct` กับ `class` ต่างกันแค่ default access** — มือใหม่บางคนคิดว่า `struct` ทำอะไร
   ไม่ได้เหมือน `class` (เช่นมี constructor หรือ private member ไม่ได้) ซึ่งไม่จริงเลย ทั้งสองคำ
   มีความสามารถเหมือนกันทุกประการ ต่างกันแค่ default access เท่านั้น

7. **สับสนระหว่าง private กับ protected เพราะยังไม่เห็นประโยชน์ชัดเจน** — ในขั้นนี้ (ก่อนเรียน
   Inheritance) หลายคนมักตั้ง field เป็น `protected` "ไว้ก่อนเผื่อใช้" ทั้งที่ยังไม่มี Class ลูกจริง
   ควรเริ่มจาก `private` เสมอ แล้วค่อยเปลี่ยนเป็น `protected` เมื่อมีเหตุผลชัดเจนจากการออกแบบ
   Inheritance จริงๆ เท่านั้น (จะพูดถึงเหตุผลนี้อย่างละเอียดใน Part 48)

---

## แบบฝึกหัดท้ายบท

1. เขียน Class `Fraction` (เศษส่วน) ที่เก็บ `numerator` (เศษ) และ `denominator` (ส่วน) เป็น
   `private` โดย `setDenominator()` ต้อง reject ค่า 0 (throw `std::invalid_argument`) เพิ่ม
   Method `toDouble()` ที่คืนค่าทศนิยมของเศษส่วนนั้น

2. Class เดิมนี้เป็น struct ที่ไม่มีการป้องกันใดๆ:
   ```cpp
   struct PersonBad { std::string name; int age; };
   ```
   ให้ Refactor เป็น Class `Person` ที่เก็บ `age` เป็น `private` พร้อม `setAge()` ที่ตรวจสอบว่า
   อายุต้องอยู่ระหว่าง 0–150 ปี และเพิ่ม Method `isAdult()` ที่คืนค่า `true` ถ้าอายุ ≥ 18

3. วิเคราะห์โค้ดต่อไปนี้ว่ามีปัญหาการออกแบบ Encapsulation อะไรบ้าง (อย่างน้อย 2 ข้อ) แล้วเสนอ
   วิธีแก้ไข:
   ```cpp
   class Circle {
   public:
       void setRadius(double r) { radius_ = r; }
       double getRadius() { return radius_; }
       void setCircumference(double c) { circumference_ = c; }
       double getCircumference() { return circumference_; }
   private:
       double radius_;
       double circumference_;
   };
   ```

4. เขียน Class `IntStack` ที่ห่อ `std::vector<int>` ไว้ภายใน โดยเปิดเผยเฉพาะ `push(int)`,
   `pop()`, `top() const`, `empty() const`, `size() const` เท่านั้น (ห้ามมี Getter ที่คืนค่า
   `vector` ภายในออกไปตรงๆ) ให้ `pop()` และ `top()` throw `std::out_of_range` เมื่อ Stack ว่าง

5. เพิ่ม `mutable` field ชื่อ `access_count_` ใน Class `Employee` จากหัวขัด 47.5 เพื่อนับว่า
   Method `name()` ถูกเรียกไปแล้วกี่ครั้ง (เพิ่มค่าทุกครั้งที่เรียก แม้ `name()` จะเป็น `const`
   method ก็ตาม) แล้วเพิ่ม Method `accessCount() const` เพื่อดูจำนวนครั้งที่เรียก

6. อธิบายด้วยคำพูดของตัวเอง (เขียนเป็นคอมเมนต์ในโค้ด หรือข้อความสั้นๆ) ว่าทำไม Class
   `BankAccount` ในหัวข้อ 47.3 ถึง **จงใจ**ไม่มี Method ชื่อ `setBalance(double)` และถ้าเพิ่ม
   Method นี้เข้าไปจะทำลายการออกแบบตรงจุดไหน

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <stdexcept>

class Fraction {
public:
    Fraction(int numerator, int denominator) : numerator_(numerator), denominator_(1) {
        setDenominator(denominator);
    }

    int numerator() const { return numerator_; }
    int denominator() const { return denominator_; }

    void setNumerator(int n) { numerator_ = n; }

    void setDenominator(int d) {
        if (d == 0) {
            throw std::invalid_argument("ตัวส่วนของเศษส่วนต้องไม่เป็น 0");
        }
        denominator_ = d;
    }

    double toDouble() const {
        return static_cast<double>(numerator_) / denominator_;
    }

    void print() const {
        std::cout << numerator_ << "/" << denominator_
                  << " = " << toDouble() << '\n';
    }

private:
    int numerator_;
    int denominator_;
};

int main() {
    Fraction half(1, 2);
    half.print();

    try {
        half.setDenominator(0);
    } catch (const std::invalid_argument& e) {
        std::cout << "ตั้งค่าไม่สำเร็จ: " << e.what() << '\n';
    }

    half.print();
    return 0;
}
```

```
1/2 = 0.5
ตั้งค่าไม่สำเร็จ: ตัวส่วนของเศษส่วนต้องไม่เป็น 0
1/2 = 0.5
```

### แนวทางเฉลยข้อ 4

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>

// ตัวอย่างการออกแบบ interface แบบ "Tell, Don't Ask": ห่อ std::vector ไว้ภายใน
// ไม่มี getter คืนค่า vector ภายในออกไปตรงๆ (ไม่งั้นคนนอกจะ push/pop มั่วได้เอง)
class IntStack {
public:
    void push(int value) { data_.push_back(value); }

    void pop() {
        if (data_.empty()) {
            throw std::out_of_range("เรียก pop() จาก stack ที่ว่างเปล่า");
        }
        data_.pop_back();
    }

    int top() const {
        if (data_.empty()) {
            throw std::out_of_range("เรียก top() จาก stack ที่ว่างเปล่า");
        }
        return data_.back();
    }

    bool empty() const { return data_.empty(); }
    std::size_t size() const { return data_.size(); }

private:
    std::vector<int> data_;
};

int main() {
    IntStack s;
    s.push(10);
    s.push(20);
    s.push(30);

    std::cout << "size=" << s.size() << " top=" << s.top() << '\n';
    s.pop();
    std::cout << "หลัง pop -> size=" << s.size() << " top=" << s.top() << '\n';

    s.pop();
    s.pop();
    try {
        s.pop();   // stack ว่างแล้ว
    } catch (const std::out_of_range& e) {
        std::cout << "pop ไม่สำเร็จ: " << e.what() << '\n';
    }

    return 0;
}
```

```
size=3 top=30
หลัง pop -> size=2 top=20
pop ไม่สำเร็จ: เรียก pop() จาก stack ที่ว่างเปล่า
```

**สังเกต**: ทั้งข้อ 1 และข้อ 4 มีจุดร่วมสำคัญเหมือนกันคือ **ทุกทางเข้าที่แก้ไข state ของ Object
ล้วนผ่านการตรวจสอบเงื่อนไข** ไม่มี "ทางลัด" ใดๆ ที่ผู้ใช้ Class จะสร้างสถานะที่ไม่ถูกต้องได้เลย
นี่คือแก่นของ Encapsulation ที่แท้จริง

(สำหรับข้อ 2, 3, 5, 6: ให้ผู้เรียนลองทำเองตามแนวทางในหัวข้อ 47.3, 47.5 และ 47.7 ของ Part นี้
โดยยึดหลัก "ทุก field ที่มี invariant ต้องมี setter ที่ validate" และ "ค่าที่คำนวณได้จากค่าอื่น
ไม่ควรเก็บเป็น field แยก" เป็นแนวทางหลักในการตัดสินใจออกแบบ)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ Access Specifier ทั้งสามระดับ (`public`, `private`, `protected`) และความแตกต่างของ
  Default Access ระหว่าง `class` กับ `struct`
- เห็นตัวอย่างจริงว่าทำไมการเปิด `public` field ตรงๆ ถึงนำไปสู่ Invalid State และ
  Encapsulation แก้ปัญหานี้ได้อย่างไรผ่าน Invariant Protection และ Information Hiding
- แยกแยะได้ระหว่าง Getter/Setter ที่ "มีประโยชน์จริง" กับ Anemic Getter/Setter ที่ไม่ได้ป้องกัน
  อะไรเลย และรู้ว่าค่าที่คำนวณได้จากค่าอื่นไม่ควรเก็บเป็น field แยก
- เข้าใจหลักการออกแบบ Public Interface แบบ **Tell, Don't Ask**
- เข้าใจ const Member Function, const-correctness, แนวคิด Immutable Object เบื้องต้น และ
  ข้อยกเว้นที่จำเป็นอย่าง `mutable`
- เห็นภาพรวมทั้งหมดผ่านตัวอย่าง `StudentGrade` ที่รวมทุกหลักการเข้าด้วยกัน

Encapsulation เป็นเสาหลักต้นแรกในสี่เสาหลักของ OOP (Encapsulation, Inheritance, Polymorphism,
Abstraction) ที่เราเพิ่งเรียนจบไป ใน **Part 48** เราจะเจาะลึกเสาหลักต้นที่สอง: **Inheritance**
ซึ่งจะทำให้เราเข้าใจความหมายที่แท้จริงของ `protected` ที่เกริ่นไว้ในหัวข้อ 47.4 อย่างถ่องแท้
พร้อมเรียนรู้การสร้างความสัมพันธ์แบบ "is-a" ระหว่าง Class การเรียก Constructor ของ Base Class
และปัญหาคลาสสิกอย่าง Diamond Problem ในกรณีของ Multiple Inheritance

**ต่อไป:** [Part 48 — Inheritance](./part-048-inheritance.md)
