# Part 48: Inheritance (Step 377–384)

> Module D — เริ่มต้น C++ และ OOP | Part 48 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 377–384
> Part ก่อนหน้า: [Part 47 — Encapsulation และ Access Specifier](./part-047-encapsulation.md) | Part ถัดไป: [Part 49 — Polymorphism และ Virtual Function](./part-049-polymorphism-virtual.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. เขียน Single Inheritance ด้วย Syntax `class Derived : public Base` และอธิบายความสัมพันธ์
   แบบ **is-a** ได้อย่างถูกต้อง
2. อธิบายลำดับการเรียก Constructor/Destructor ระหว่าง Base และ Derived Class และเรียก
   Constructor ของ Base ผ่าน Initializer List ได้อย่างถูกต้อง
3. เข้าใจว่าทำไม Member ที่เป็น `private` ใน Base Class ถึงเข้าถึงไม่ได้จาก Derived Class
   และรู้ว่าเมื่อไหร่ควรใช้ `protected` แทน พร้อมข้อควรระวัง
4. แยกแยะความหมายของ Inheritance แบบ `public`, `protected`, `private` ได้
5. เขียนและเข้าใจ Multilevel Inheritance (สืบทอดหลายชั้น) และ Multiple Inheritance
   (สืบทอดจากหลาย Class พร้อมกัน)
6. อธิบาย **Diamond Problem** ที่เกิดจาก Multiple Inheritance ได้ พร้อมแก้ไขด้วย
   **Virtual Inheritance**
7. ตัดสินใจได้ว่าเมื่อไหร่ควรใช้ Inheritance และเมื่อไหร่ควรใช้ **Composition** แทน
   ตามหลัก "Composition over Inheritance"

---

## 48.1 Single Inheritance: is-a Relationship (Step 377)

**Inheritance (การสืบทอด)** คือกลไกที่ให้ Class หนึ่ง (เรียกว่า **Derived Class** หรือ Class ลูก)
สามารถ "รับ" Member Variable และ Member Function ทั้งหมดจาก Class อีกตัวหนึ่ง (เรียกว่า
**Base Class** หรือ Class แม่) มาใช้งานได้ทันที โดยไม่ต้องเขียนโค้ดซ้ำ

ความสัมพันธ์นี้เรียกว่า **is-a** (Derived "เป็น" Base ชนิดหนึ่ง) เช่น "Car เป็น Vehicle ชนิดหนึ่ง"
(Car is a Vehicle) — นี่คือกฎทองในการตัดสินใจว่าควรใช้ Inheritance หรือไม่: **ถ้าประโยค
"X is a Y" ฟังแล้วสมเหตุสมผลตามธรรมชาติ Inheritance คือตัวเลือกที่เหมาะสม**

### Syntax

```cpp
class Derived : public Base {
    // ...
};
```

`: public Base` หมายถึง "Derived สืบทอดจาก Base แบบ public" (เราจะพูดถึงความหมายของคำว่า
`public` ตรงนี้อย่างละเอียดในหัวข้อ 48.4 — ตอนนี้ให้จำไว้ก่อนว่านี่คือรูปแบบที่ใช้บ่อยที่สุด)

```cpp
#include <iostream>
#include <string>

// Base class: Vehicle
class Vehicle {
public:
    explicit Vehicle(std::string brand) : brand_(std::move(brand)) {}

    void honk() const { std::cout << brand_ << ": ปี๊น ปี๊น!\n"; }
    const std::string& brand() const { return brand_; }

private:
    std::string brand_;
};

// Derived class: Car "is-a" Vehicle
// class Car : public Vehicle  ->  Car สืบทอดคุณสมบัติทั้งหมดของ Vehicle มา
class Car : public Vehicle {
public:
    Car(std::string brand, int numDoors) : Vehicle(std::move(brand)), numDoors_(numDoors) {}

    void openTrunk() const {
        std::cout << brand() << ": เปิดท้ายรถแล้ว (มี " << numDoors_ << " ประตู)\n";
    }

private:
    int numDoors_;
};

int main() {
    Car myCar("Toyota", 4);

    myCar.honk();        // honk() มาจาก Vehicle แต่ Car เรียกใช้ได้เลยโดยไม่ต้องเขียนซ้ำ
    myCar.openTrunk();   // openTrunk() เป็น method ใหม่ที่มีเฉพาะใน Car

    // Car "is-a" Vehicle: Car* หรือ Car& สามารถใช้ในที่ที่ต้องการ Vehicle ได้
    const Vehicle& v = myCar;
    std::cout << "ในฐานะ Vehicle ทั่วไป ยี่ห้อคือ: " << v.brand() << '\n';

    return 0;
}
```

```
Toyota: ปี๊น ปี๊น!
Toyota: เปิดท้ายรถแล้ว (มี 4 ประตู)
ในฐานะ Vehicle ทั่วไป ยี่ห้อคือ: Toyota
```

สังเกตสามจุดสำคัญ:

1. `Car` ไม่ได้ประกาศ `honk()` หรือ `brand()` เลย แต่เรียกใช้ได้ทันที เพราะสืบทอดมาจาก `Vehicle`
2. `Car` เพิ่ม Member ใหม่ของตัวเอง (`numDoors_`, `openTrunk()`) ที่ `Vehicle` ไม่มี
3. Object ของ `Car` สามารถใช้แทน `Vehicle` ได้ในทุกที่ที่ต้องการ `Vehicle&` หรือ `Vehicle*`
   (เรียกว่า **Substitutability** — คุณสมบัตินี้จะสำคัญมากขึ้นเมื่อเรียนเรื่อง Polymorphism ใน
   Part 49)

---

## 48.2 การเรียก Constructor ของ Base Class ผ่าน Initializer List (Step 378)

เมื่อสร้าง Object ของ Derived Class, **Constructor ของ Base Class จะถูกเรียกก่อนเสมอ**
โดยอัตโนมัติ ก่อนที่ Body ของ Constructor ของ Derived Class จะเริ่มทำงาน และเมื่อ Object
ถูกทำลาย **Destructor จะทำงานในลำดับย้อนกลับ**: Derived ก่อน แล้วค่อย Base

ถ้า Base Class ไม่มี Default Constructor (Constructor ที่ไม่รับ Argument) เราต้อง **ระบุ
Argument ให้ Base Constructor อย่างชัดเจนผ่าน Initializer List** ของ Derived Class เท่านั้น
(เรียกใน Body ของ Constructor ไม่ได้ เหมือนที่เรียนเรื่อง Initializer List มาแล้วใน Part 46)

```cpp
#include <iostream>
#include <string>

class Base {
public:
    explicit Base(const std::string& tag) : tag_(tag) {
        std::cout << "  Base(" << tag_ << ") constructor\n";
    }
    ~Base() { std::cout << "  ~Base(" << tag_ << ") destructor\n"; }

private:
    std::string tag_;
};

class Derived : public Base {
public:
    explicit Derived(const std::string& tag) : Base(tag), tag_(tag) {
        // Base(tag) ต้องถูกเรียกใน initializer list ของ Derived เท่านั้น
        // จะเรียก Base::Base() ใน body ของ constructor ไม่ได้เลย
        std::cout << "  Derived(" << tag_ << ") constructor\n";
    }
    ~Derived() { std::cout << "  ~Derived(" << tag_ << ") destructor\n"; }

private:
    std::string tag_;
};

int main() {
    std::cout << "สร้าง Derived object:\n";
    {
        Derived d("A");
        std::cout << "-- ใช้งาน d --\n";
    }   // d ออกจาก scope ที่นี่ -> destructor ทำงาน
    std::cout << "จบโปรแกรม\n";
    return 0;
}
```

```
สร้าง Derived object:
  Base(A) constructor
  Derived(A) constructor
-- ใช้งาน d --
  ~Derived(A) destructor
  ~Base(A) destructor
จบโปรแกรม
```

ลำดับนี้สมเหตุสมผลมาก: Derived Class "ขยาย" Base Class ดังนั้น Base ต้อง "พร้อมใช้งานสมบูรณ์"
ก่อนที่ Derived จะเริ่มเพิ่มเติมส่วนของตัวเองได้ และตอนทำลาย Object ก็ต้อง "ทำลายส่วนที่สร้างทีหลัง
ก่อน" เพื่อไม่ให้ Derived ใช้งาน Base ที่ถูกทำลายไปแล้วโดยไม่ได้ตั้งใจ — เป็นหลักการเดียวกับ RAII
ที่เรียนใน Part 46

### ถ้าลืมเรียก Base Constructor ที่จำเป็น

```cpp
#include <string>

class Base {
public:
    explicit Base(int value) : value_(value) {}
private:
    int value_;
};

class Derived : public Base {
public:
    Derived() {}   // ผิด: ไม่ได้เรียก Base(int) และ Base ไม่มี default constructor
private:
    int extra_ = 0;
};

int main() {
    Derived d;
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 no_default_ctor.cpp -o no_default_ctor
```

```
no_default_ctor.cpp: In constructor 'Derived::Derived()':
no_default_ctor.cpp:12:15: error: no matching function for call to 'Base::Base()'
   12 |     Derived() {}
      |               ^
no_default_ctor.cpp:5:14: note: candidate: 'Base::Base(int)'
    5 |     explicit Base(int value) : value_(value) {}
      |              ^~~~
no_default_ctor.cpp:5:14: note:   candidate expects 1 argument, 0 provided
```

Compiler พยายามเรียก `Base()` (Default Constructor) ให้อัตโนมัติเมื่อ Derived Constructor ไม่ได้
ระบุไว้เอง แต่เพราะ `Base` มีแค่ `Base(int)` เท่านั้น (การประกาศ Constructor แบบมี Parameter
ทำให้ Compiler ไม่สร้าง Default Constructor ให้อัตโนมัติอีกต่อไป อย่างที่เรียนมาใน Part 46)
จึง Compile Error วิธีแก้คือต้องเรียก `Base(ค่าบางอย่าง)` ให้ชัดเจนใน Initializer List ของ
`Derived`

---

## 48.3 protected ในบริบทของ Inheritance (Step 379)

ใน Part 47 เราเกริ่นไว้ว่า `protected` คล้าย `private` แต่เปิดให้ Derived Class เข้าถึงได้
ตอนนี้เราจะเห็นเหตุผลที่แท้จริงว่าทำไมถึงต้องมี Access Specifier ตัวที่สามนี้

### private ใน Base Class เข้าถึงไม่ได้จาก Derived Class

```cpp
#include <iostream>
#include <string>

class Animal {
public:
    explicit Animal(std::string name) : name_(std::move(name)) {}

private:
    std::string name_;   // private: Derived class เข้าถึงไม่ได้ แม้จะสืบทอดมาก็ตาม
};

class Dog : public Animal {
public:
    explicit Dog(std::string name) : Animal(std::move(name)) {}

    void bark() const {
        std::cout << name_ << ": โฮ่ง!\n";   // ผิด: name_ เป็น private ของ Animal เข้าถึงไม่ได้
    }
};

int main() {
    Dog d("โปงลาง");
    d.bark();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 private_not_accessible.cpp -o private_not_accessible
```

```
private_not_accessible.cpp: In member function 'void Dog::bark() const':
private_not_accessible.cpp:17:22: error: 'std::string Animal::name_' is private within this context
   17 |         std::cout << name_ << ": โฮ่ง!\n";
      |                      ^~~~~
private_not_accessible.cpp:9:17: note: declared private here
    9 |     std::string name_;
      |                 ^~~~~
```

แม้ `Dog` จะ "สืบทอด" `name_` มาจาก `Animal` จริง (พูดให้แม่นยำคือ Object ของ `Dog` มี
`name_` ฝังอยู่ข้างในจริงๆ) แต่ **โค้ดภายใน `Dog` เองก็ยังเข้าถึง `name_` โดยตรงไม่ได้**
เพราะ `private` จำกัดสิทธิ์ไว้เฉพาะโค้ดภายใน `Animal` เท่านั้น ไม่ว่า Class อื่นจะสืบทอดมาหรือไม่
ก็ตาม

### แก้ด้วย protected

```cpp
#include <iostream>
#include <string>

class Animal {
public:
    explicit Animal(std::string name) : name_(std::move(name)) {}

    const std::string& name() const { return name_; }   // ทางเลือก: accessor method (แนะนำ)

protected:
    std::string name_;   // protected: Derived class เข้าถึงได้โดยตรง
};

class Dog : public Animal {
public:
    explicit Dog(std::string name) : Animal(std::move(name)) {}

    void bark() const {
        // เข้าถึง name_ ได้โดยตรงเพราะเป็น protected (Dog เป็น class ลูกของ Animal)
        std::cout << name_ << ": โฮ่ง!\n";
    }
};

int main() {
    Dog d("โปงลาง");
    d.bark();

    // ยังคงใช้ accessor method name() ได้ตามปกติจากภายนอกด้วย
    std::cout << "ชื่อผ่าน accessor: " << d.name() << '\n';

    // d.name_ = "ใหม่";  // ผิด: name_ เป็น protected เข้าถึงจากภายนอก class ไม่ได้
    return 0;
}
```

```
โปงลาง: โฮ่ง!
ชื่อผ่าน accessor: โปงลาง
```

### ข้อควรระวังเรื่อง protected

แม้ `protected` จะแก้ปัญหาข้างต้นได้ แต่ **นักออกแบบ C++ มืออาชีพจำนวนมากใช้ `protected`
อย่างระมัดระวังมาก** ด้วยเหตุผลนี้:

> `protected` คือการ "แหก" Encapsulation ออกไปให้ Derived Class ทุกตัวในอนาคต — ซึ่งอาจมี
> จำนวนมากและเขียนโดยคนละคนกัน ถ้า `Animal` เปลี่ยนวิธีเก็บ `name_` ภายในทีหลัง (เช่นเปลี่ยน
> เป็น `std::string* ` หรือรวมกับ field อื่น) **Derived Class ทุกตัวที่เข้าถึง `name_` โดยตรง
> จะพังทันที** ในขณะที่ถ้าใช้ `private` + accessor method (`name()`) เหมือนใน Part 47
> Base Class จะเปลี่ยน Implementation ภายในได้อย่างอิสระ โดย Derived Class ไม่ต้องรู้เลย

**แนวทางที่แนะนำ**: เริ่มต้นด้วย `private` เสมอ และเปิด Accessor Method (`protected` หรือ
`public` ก็ได้แล้วแต่กรณี) ให้ Derived Class ใช้แทนการเข้าถึง Field ตรงๆ ใช้ `protected` กับ
ตัวแปรโดยตรงเฉพาะเมื่อมีเหตุผลจำเป็นจริงๆ เช่นต้องการ Performance สูงสุดหรือ Derived Class
ต้องแก้ไขค่านั้นบ่อยมาก

---

## 48.4 Access Specifier ของการสืบทอด: public, protected, private Inheritance (Step 380)

นอกจาก Access Specifier ของ Member แล้ว **การสืบทอดเองก็มี Access Specifier ด้วย** ซึ่งกำหนด
ว่า Member ของ Base Class จะ "เปลี่ยนระดับการเข้าถึง" อย่างไรเมื่อถูกมองจาก Derived Class

| รูปแบบการสืบทอด | `public` ของ Base กลายเป็น | `protected` ของ Base กลายเป็น | ความหมาย |
|---|---|---|---|
| `class D : public B` | `public` | `protected` | is-a (พบบ่อยที่สุด 99% ของเวลา) |
| `class D : protected B` | `protected` | `protected` | ใช้น้อยมาก |
| `class D : private B` | `private` | `private` | implemented-in-terms-of (ใช้น้อยมาก) |

```cpp
#include <iostream>

class Base {
public:
    int pub = 1;
protected:
    int prot = 2;
private:
    int priv = 3;
};

// public inheritance: public ของ Base ยังเป็น public ใน Derived, protected ยังเป็น protected
// (นี่คือรูปแบบที่ใช้ 99% ของเวลาในโค้ดจริง เพราะสื่อความหมาย "is-a" ตรงไปตรงมา)
class PublicDerived : public Base {
public:
    void show() const {
        std::cout << "pub=" << pub << " prot=" << prot << '\n';
        // priv เข้าถึงไม่ได้ไม่ว่า inherit แบบไหนก็ตาม
    }
};

// private inheritance: public และ protected ของ Base กลายเป็น private ทั้งหมดใน Derived
// ความหมายคือ "implemented-in-terms-of" ไม่ใช่ "is-a" — ใช้น้อยมากในโค้ดจริง
class PrivateDerived : private Base {
public:
    void show() const {
        std::cout << "pub=" << pub << " prot=" << prot << '\n';
    }
};

int main() {
    PublicDerived pd;
    std::cout << "pd.pub เข้าถึงจากภายนอกได้: " << pd.pub << '\n';
    pd.show();

    PrivateDerived prd;
    // prd.pub เข้าถึงจากภายนอกไม่ได้ เพราะ private inheritance ทำให้ pub กลายเป็น private ใน PrivateDerived
    prd.show();

    return 0;
}
```

```
pd.pub เข้าถึงจากภายนอกได้: 1
pub=1 prot=2
pub=1 prot=2
```

> **หมายเหตุ**: `class` มี Default Inheritance Access เป็น `private` (ถ้าไม่ระบุคำใดเลย เช่น
> `class D : Base {}` จะเท่ากับ `private` โดยอัตโนมัติ) ในขณะที่ `struct` มี Default เป็น
> `public` — เหมือนกับ Default Access ของ Member ที่เรียนใน Part 47 ทุกประการ แต่เพื่อความชัดเจน
> ตลอดหลักสูตรนี้เราจะระบุ `public`/`protected`/`private` ให้ชัดเจนเสมอ ไม่พึ่งพา Default

ตลอดหลักสูตรนี้ เราจะใช้ **`public` Inheritance แทบทั้งหมด** เพราะมันคือรูปแบบที่สื่อความหมาย
is-a ได้ตรงไปตรงมาที่สุด และเป็นรูปแบบเดียวที่ทำให้ Polymorphism (Part 49) ทำงานได้ตามที่คาดหวัง

---

## 48.5 Multilevel Inheritance: สืบทอดหลายชั้น (Step 381)

Inheritance สามารถต่อกันเป็นสายโซ่ได้หลายชั้น เรียกว่า **Multilevel Inheritance**:
`LivingBeing` → `Animal` → `Dog`

```cpp
#include <iostream>
#include <string>

// Level 1
class LivingBeing {
public:
    explicit LivingBeing(std::string name) : name_(std::move(name)) {
        std::cout << "  LivingBeing(" << name_ << ")\n";
    }
    const std::string& name() const { return name_; }

private:
    std::string name_;
};

// Level 2: Animal is-a LivingBeing
class Animal : public LivingBeing {
public:
    Animal(std::string name, int legCount)
        : LivingBeing(std::move(name)), legCount_(legCount) {
        std::cout << "  Animal(legCount=" << legCount_ << ")\n";
    }
    int legCount() const { return legCount_; }

private:
    int legCount_;
};

// Level 3: Dog is-a Animal (และโดยอ้อมคือ is-a LivingBeing ด้วย)
class Dog : public Animal {
public:
    explicit Dog(std::string name) : Animal(std::move(name), 4) {
        std::cout << "  Dog()\n";
    }

    void describe() const {
        // เรียกใช้ method จากทั้ง 3 ระดับได้หมด: name() จาก LivingBeing, legCount() จาก Animal
        std::cout << name() << " เป็นสัตว์ที่มี " << legCount() << " ขา\n";
    }
};

int main() {
    std::cout << "สร้าง Dog:\n";
    Dog d("โปงลาง");
    d.describe();
    return 0;
}
```

```
สร้าง Dog:
  LivingBeing(โปงลาง)
  Animal(legCount=4)
  Dog()
โปงลาง เป็นสัตว์ที่มี 4 ขา
```

สังเกตลำดับการเรียก Constructor: **จากรากสายโซ่ไปหาปลายสายโซ่เสมอ** (`LivingBeing` →
`Animal` → `Dog`) และ `Dog` เรียก `Animal(name, 4)` ใน Initializer List ของตัวเอง โดย `Animal`
เป็นคนเรียก `LivingBeing(name)` ต่ออีกที — `Dog` **ไม่สามารถ**ข้าม `Animal` ไปเรียก
`LivingBeing` ตรงๆ ได้เลย ต้องผ่าน `Animal` เสมอ (ยกเว้นกรณี Virtual Inheritance ที่จะเรียนใน
หัวข้อถัดไป)

---

## 48.6 Multiple Inheritance: สืบทอดจากหลาย Class พร้อมกัน (Step 382)

C++ อนุญาตให้ Class หนึ่งสืบทอดจาก **หลาย Base Class พร้อมกัน** ได้ (ต่างจากภาษาอย่าง Java
หรือ C# ที่อนุญาตให้สืบทอด Class ได้แค่ตัวเดียว) เรียกว่า **Multiple Inheritance**

```cpp
#include <iostream>
#include <string>

class Flyable {
public:
    explicit Flyable(double maxAltitudeM) : maxAltitudeM_(maxAltitudeM) {}
    void fly() const {
        std::cout << "บินได้สูงสุด " << maxAltitudeM_ << " เมตร\n";
    }
private:
    double maxAltitudeM_;
};

class Swimmable {
public:
    explicit Swimmable(double maxDepthM) : maxDepthM_(maxDepthM) {}
    void swim() const {
        std::cout << "ดำน้ำได้ลึกสุด " << maxDepthM_ << " เมตร\n";
    }
private:
    double maxDepthM_;
};

// Duck สืบทอดจากสอง class พร้อมกัน (Multiple Inheritance)
class Duck : public Flyable, public Swimmable {
public:
    Duck(std::string name, double maxAltitudeM, double maxDepthM)
        : Flyable(maxAltitudeM), Swimmable(maxDepthM), name_(std::move(name)) {}

    void showAbilities() const {
        std::cout << name_ << " ทำได้ทั้งสองอย่าง:\n  ";
        fly();     // มาจาก Flyable
        std::cout << "  ";
        swim();    // มาจาก Swimmable
    }

private:
    std::string name_;
};

int main() {
    Duck d("โดนัลด์", 500.0, 3.0);
    d.showAbilities();
    return 0;
}
```

```
โดนัลด์ ทำได้ทั้งสองอย่าง:
  บินได้สูงสุด 500 เมตร
  ดำน้ำได้ลึกสุด 3 เมตร
```

Syntax คือคั่นแต่ละ Base Class ด้วย comma: `class Duck : public Flyable, public Swimmable`
และต้องเรียก Constructor ของ **ทุก** Base Class ใน Initializer List (`Flyable(maxAltitudeM),
Swimmable(maxDepthM)`) ลำดับการเรียก Constructor จะเป็นไปตาม **ลำดับที่ประกาศไว้ตอนสืบทอด**
(`Flyable` ก่อน `Swimmable`) ไม่ใช่ลำดับที่เขียนใน Initializer List

Multiple Inheritance มีประโยชน์เมื่อต้องการรวม "ความสามารถ" (capability) ที่เป็นอิสระต่อกัน
เข้าด้วยกัน — แต่ก็เป็นที่มาของปัญหาคลาสสิกที่เราจะพูดถึงต่อไป

---

## 48.7 Diamond Problem และการแก้ไขด้วย Virtual Inheritance (Step 383)

เมื่อ Base Class สองตัวที่ถูกสืบทอดพร้อมกันดันมี **Base Class ร่วมกันอีกทีหนึ่ง** จะเกิดปัญหาที่
เรียกว่า **Diamond Problem** ชื่อนี้มาจากรูปร่างของแผนภาพการสืบทอดที่มีลักษณะเป็นข้าวหลามตัด:

```
              Animal
             /      \
       Flying        Swimming
             \      /
           FlyingFish
```

`FlyingFish` สืบทอดทั้ง `Flying` และ `Swimming` ซึ่งทั้งคู่สืบทอดจาก `Animal` อีกที ปัญหาคือ
**`FlyingFish` จะมี Animal subobject อยู่สองชุดซ้อนกัน** (หนึ่งชุดผ่าน `Flying`, อีกหนึ่งชุด
ผ่าน `Swimming`) ทำให้เกิดความกำกวมทันทีที่พยายามเข้าถึง Member ที่มาจาก `Animal`:

```cpp
#include <iostream>
#include <string>

class Animal {
public:
    explicit Animal(std::string name) : name_(std::move(name)) {}
    const std::string& name() const { return name_; }
protected:
    std::string name_;
};

// Flying และ Swimming ต่างก็สืบทอดจาก Animal (public ธรรมดา ไม่ใช่ virtual)
class Flying : public Animal {
public:
    explicit Flying(std::string name) : Animal(std::move(name)) {}
};

class Swimming : public Animal {
public:
    explicit Swimming(std::string name) : Animal(std::move(name)) {}
};

// FlyingFish สืบทอดจากทั้ง Flying และ Swimming
// -> ปัญหา: FlyingFish จะมี "สอง copy" ของ Animal ซ้อนกันอยู่ (Diamond Problem)
class FlyingFish : public Flying, public Swimming {
public:
    FlyingFish(std::string name)
        : Flying(name), Swimming(name) {}
};

int main() {
    FlyingFish f("Nemo");

    std::cout << f.name() << '\n';   // Error: ambiguous - name() มาจาก Animal ทาง Flying หรือ Swimming?

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 diamond_problem.cpp -o diamond_problem
```

```
diamond_problem.cpp: In function 'int main()':
diamond_problem.cpp:34:20: error: request for member 'name' is ambiguous
   34 |     std::cout << f.name() << '\n';
      |                    ^~~~
diamond_problem.cpp:7:24: note: candidates are: 'const std::string& Animal::name() const'
diamond_problem.cpp:7:24: note:                 'const std::string& Animal::name() const'
```

Compiler บอกตรงๆ ว่า "candidates are: ... name() ... name()" (สองตัวเลือกที่หน้าตาเหมือนกัน
เป๊ะ) เพราะมี `Animal::name()` อยู่สองชุดจริงๆ ใน `FlyingFish` หนึ่งชุดจาก `Flying::Animal`
และอีกชุดจาก `Swimming::Animal`

### แก้ไขด้วย Virtual Inheritance

C++ แก้ปัญหานี้ด้วย **Virtual Inheritance**: การเติมคำว่า `virtual` ตอนสืบทอด Base Class
บอก Compiler ว่า "ถ้ามีหลายเส้นทางสืบทอดมาถึง Base Class ตัวนี้ ให้แชร์ subobject เดียวกัน
ไม่ต้องสร้างซ้ำ"

```cpp
#include <iostream>
#include <string>

class Animal {
public:
    explicit Animal(std::string name) : name_(std::move(name)) {}
    const std::string& name() const { return name_; }
protected:
    std::string name_;
};

// virtual inheritance: บอกคอมไพเลอร์ว่า "Animal ที่สืบทอดผ่านทางนี้ ให้แชร์กันเป็น subobject เดียว"
class Flying : public virtual Animal {
public:
    explicit Flying(std::string name) : Animal(std::move(name)) {}
    void fly() const { std::cout << name() << " กำลังบิน\n"; }
};

class Swimming : public virtual Animal {
public:
    explicit Swimming(std::string name) : Animal(std::move(name)) {}
    void swim() const { std::cout << name() << " กำลังว่ายน้ำ\n"; }
};

// เมื่อทั้ง Flying และ Swimming สืบทอด Animal แบบ virtual แล้ว
// FlyingFish จะมี Animal subobject เดียวเท่านั้น -> name() ไม่กำกวมอีกต่อไป
class FlyingFish : public Flying, public Swimming {
public:
    // ในกรณี virtual inheritance: class ที่อยู่ล่างสุด (most-derived class) ต้อง
    // เรียก constructor ของ virtual base (Animal) เองโดยตรงเสมอ ไม่ผ่าน Flying/Swimming
    explicit FlyingFish(std::string name)
        : Animal(name), Flying(name), Swimming(name) {}
};

int main() {
    FlyingFish f("Nemo");

    std::cout << f.name() << " ทำได้สองอย่าง:\n";  // ไม่กำกวมแล้ว เพราะมี Animal เดียว
    f.fly();
    f.swim();

    return 0;
}
```

```
Nemo ทำได้สองอย่าง:
Nemo กำลังบิน
Nemo กำลังว่ายน้ำ
```

จุดสำคัญที่ต้องจำเกี่ยวกับ Virtual Inheritance:

1. **ประกาศ `virtual` ที่จุดสืบทอดของ Base Class ตัวกลาง** (`Flying`, `Swimming`) ไม่ใช่ที่
   `FlyingFish`
2. **Class ที่อยู่ล่างสุดของสายโซ่ (Most-Derived Class)** — ในที่นี้คือ `FlyingFish` —
   **ต้องเรียก Constructor ของ Virtual Base Class (`Animal`) เองโดยตรง** ใน Initializer List
   เสมอ แม้ว่า `Flying` และ `Swimming` จะเรียก `Animal(...)` ไว้ในตัวเองแล้วก็ตาม เพราะกฎของ
   C++ กำหนดให้ Virtual Base ถูก Initialize โดย Most-Derived Class เท่านั้น (ถ้าไม่ระบุ
   Compiler จะเรียก Default Constructor ของ `Animal` ให้อัตโนมัติ ซึ่งจะ Error ถ้า `Animal`
   ไม่มี Default Constructor)
3. ค่าที่ `Flying(name)` และ `Swimming(name)` ส่งให้ `Animal` **จะถูกมองข้ามไปเลย** เพราะ
   Most-Derived Class เป็นคนตัดสินใจค่าสุดท้ายที่ใช้ Initialize Virtual Base — ในตัวอย่างนี้
   `Animal(name)` ที่ `FlyingFish` เรียกเองคือค่าที่ถูกใช้จริง

> **ในทางปฏิบัติ**: Diamond Problem และ Virtual Inheritance เป็นเรื่องที่พบได้จริงในโค้ดขนาดใหญ่
> แต่หลายทีมเลือกที่จะ **หลีกเลี่ยง Multiple Inheritance ของ Class ที่มี State (ข้อมูล) เลย**
> และสงวน Multiple Inheritance ไว้สำหรับ Abstract Class ที่ไม่มี Data Member เลย (เรียกว่า
> "Interface" ในความหมายแบบภาษาอื่น) ซึ่งจะเรียนอย่างละเอียดใน **Part 50 — Abstract Class
> และ Interface** เพราะ Interface ที่ไม่มี State จะไม่มีทางเกิด Diamond Problem ได้เลยตั้งแต่ต้น

---

## 48.8 is-a vs has-a: Composition over Inheritance (Step 384)

Inheritance เป็นเครื่องมือที่ทรงพลัง แต่ **ไม่ใช่คำตอบสำหรับทุกปัญหาการ Reuse โค้ด** หลักการ
สำคัญที่วิศวกรซอฟต์แวร์มืออาชีพยึดถือคือ:

> **"Composition over Inheritance"** — ให้พิจารณาใช้ Composition (การรวม Object เป็น Member
> ของอีก Object หนึ่ง หรือความสัมพันธ์แบบ **has-a**) ก่อนเสมอ แล้วค่อยใช้ Inheritance เมื่อ
> ความสัมพันธ์แบบ is-a สมเหตุสมผลจริงๆ และต้องการ Substitutability (ใช้แทนกันได้ผ่าน Base
> Pointer/Reference)

### ตัวอย่างการใช้ Inheritance "ผิดที่"

ปัญหาคลาสสิกคือการใช้ Inheritance แค่เพื่อ "ยืม" ฟังก์ชันจาก Class อื่นมาใช้ โดยไม่ได้คำนึงว่า
ความสัมพันธ์แบบ is-a นั้นสมเหตุสมผลจริงหรือไม่:

```cpp
#include <iostream>
#include <vector>

// ตัวอย่างการใช้ inheritance "ผิดที่": StackBad สืบทอดจาก std::vector<int>
// เพื่อหวัง "reuse" push_back/pop_back แต่ผลคือ StackBad กลายเป็น vector ไปโดยปริยาย
class StackBad : public std::vector<int> {
public:
    void push(int v) { push_back(v); }
    void pop() { pop_back(); }
    int top() const { return back(); }
};

int main() {
    StackBad s;
    s.push(1);
    s.push(2);
    s.push(3);

    std::cout << "top=" << s.top() << '\n';

    // ปัญหา: เพราะ StackBad "is-a" vector<int> (ตาม type system) คนใช้จึงเรียก method
    // ของ vector ที่ไม่ควรมีใน Stack ได้ตรงๆ เช่น สอดแทรกค่ากลาง stack, เข้าถึงสมาชิกด้วย index
    s.insert(s.begin(), 999);         // ผิดหลักการของ Stack (LIFO) แต่คอมไพล์ผ่านสนิท!
    std::cout << "s[0]=" << s[0] << " (ทำลายกฎ LIFO ของ Stack ไปแล้ว)\n";

    return 0;
}
```

```
top=3
s[0]=999 (ทำลายกฎ LIFO ของ Stack ไปแล้ว)
```

ปัญหาคือ **"Stack is-a vector" ไม่สมเหตุสมผลตามหลักการออกแบบที่ดี** แม้ Stack จะ *implement*
ด้วย vector ภายในได้ก็ตาม เพราะ Stack ควรมีกฎ LIFO (Last-In-First-Out) ที่เข้มงวด แต่การสืบทอด
จาก `std::vector` ทำให้ผู้ใช้เรียก Method อะไรก็ได้ของ vector ผ่าน Object ของ Stack ได้ตรงๆ
ทำลาย Invariant ที่ Stack ควรจะรักษาไว้ทั้งหมด (นี่คือตัวอย่างของการละเมิดหลักการที่เรียกว่า
**Liskov Substitution Principle** ซึ่งจะเรียนอย่างละเอียดใน Part 80 — Modern C++ Best Practice)

### ออกแบบใหม่ด้วย Composition

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>

// ตัวอย่างการใช้ composition แทน inheritance: Stack "has-a" vector<int> เป็น private member
// ไม่ใช่ "is-a" vector -> เปิดเผยเฉพาะ push/pop/top ที่สอดคล้องกับความหมายของ Stack เท่านั้น
class Stack {
public:
    void push(int v) { data_.push_back(v); }

    void pop() {
        if (data_.empty()) throw std::out_of_range("pop จาก stack ที่ว่างเปล่า");
        data_.pop_back();
    }

    int top() const {
        if (data_.empty()) throw std::out_of_range("top จาก stack ที่ว่างเปล่า");
        return data_.back();
    }

    bool empty() const { return data_.empty(); }

private:
    std::vector<int> data_;   // has-a: Stack "มี" vector อยู่ภายใน ไม่ใช่ "เป็น" vector
};

int main() {
    Stack s;
    s.push(1);
    s.push(2);
    s.push(3);

    std::cout << "top=" << s.top() << '\n';

    // s.insert(...)  // ไม่มี method นี้ให้เรียกเลย เพราะ Stack ไม่ได้ "is-a" vector
    // Public Interface ของ Stack ถูกจำกัดไว้เฉพาะพฤติกรรมที่ถูกต้องตามหลัก LIFO เท่านั้น

    s.pop();
    std::cout << "หลัง pop -> top=" << s.top() << '\n';

    return 0;
}
```

```
top=3
หลัง pop -> top=2
```

ตอนนี้ `Stack` เก็บ `std::vector<int>` เป็น **private member** (Composition / has-a) แทนที่จะ
สืบทอดจากมัน Public Interface ของ `Stack` จึงมีแค่ `push()`, `pop()`, `top()`, `empty()`
เท่านั้น — ผู้ใช้ไม่มีทางเรียก Method ที่ทำลายกฎ LIFO ได้เลย เพราะ Method เหล่านั้นไม่ได้ถูก
เปิดเผยออกมาตั้งแต่แรก (Encapsulation ที่เรียนใน Part 47 ทำงานร่วมกับหลักการนี้อย่างสมบูรณ์)

### เมื่อไหร่ควรใช้ Inheritance vs เมื่อไหร่ควรใช้ Composition

| สถานการณ์ | ควรใช้ |
|---|---|
| ประโยค "X is a Y" ฟังแล้วสมเหตุสมผลตามธรรมชาติ และต้องการใช้ X แทน Y ได้ผ่าน pointer/reference | **Inheritance** |
| ต้องการแค่ "ยืม" ฟังก์ชันบางส่วนมาใช้ โดยไม่ต้องการความสัมพันธ์แบบ is-a จริงๆ | **Composition** |
| Derived Class ต้องเปลี่ยนพฤติกรรมบางอย่างของ Base อย่างสิ้นเชิงจนขัดกับสัญญาเดิม (violate invariant ของ Base) | **Composition** (สัญญาณว่า is-a ไม่จริง) |
| ต้องการเปลี่ยน "พฤติกรรมภายใน" ของ Object ได้แบบ Runtime (สลับ Algorithm ไปมา) | **Composition** (เรียกว่า Strategy Pattern จะเรียนใน Part 98) |
| ต้องการให้หลาย Class ที่ไม่เกี่ยวข้องกันมีบาง "ความสามารถ" ร่วมกัน โดยไม่มี State ร่วมกัน | **Multiple Inheritance ของ Interface** (เรียนเจาะลึกใน Part 50) |

กฎง่ายๆ ที่ใช้ได้เสมอ: **ก่อนเขียน `class Derived : public Base` ให้ถามตัวเองก่อนว่า "Derived
เป็น Base จริงๆ หรือแค่ Derived อยากใช้ของบางอย่างที่ Base มี?"** ถ้าเป็นแบบหลัง ให้ใช้
Composition (เก็บ Base เป็น Member Variable) เสมอ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมเรียก Base Constructor ที่ต้องการ Argument** — ถ้า Base Class ไม่มี Default
   Constructor และ Derived Class ไม่ได้ระบุ Base Constructor ใน Initializer List อย่างชัดเจน
   จะ Compile Error ทันที (ดูตัวอย่างในหัวข้อ 48.2)

2. **พยายามเข้าถึง private Member ของ Base จาก Derived โดยตรง** — ต้องใช้ `protected` หรือ
   Accessor Method (`public`) แทน ไม่มีทางเข้าถึง `private` Member ของ Base จาก Derived ได้เลย
   ไม่ว่ากรณีใดก็ตาม (ดูหัวข้อ 48.3)

3. **ใช้ `protected` พร่ำเพรื่อ "เผื่อไว้ก่อน"** — ทำให้ Encapsulation อ่อนแอลงโดยไม่จำเป็น
   ควรเริ่มจาก `private` เสมอ แล้วเปลี่ยนเป็น `protected` เฉพาะเมื่อมีเหตุผลจากการออกแบบจริงๆ

4. **ใช้ Multiple Inheritance กับ Class ที่มี State ซ้อนกันโดยไม่คำนึงถึง Diamond Problem** —
   นำไปสู่ Compile Error แบบกำกวม (ambiguous) หรือถ้าไม่ระวังอาจทำให้เกิด Object ที่มีข้อมูล
   ซ้อนกันโดยไม่ตั้งใจ วิธีที่ปลอดภัยที่สุดคือหลีกเลี่ยง Multiple Inheritance ของ Class ที่มี
   State เว้นแต่จำเป็นจริงๆ และเข้าใจ Virtual Inheritance อย่างถ่องแท้ก่อนใช้

5. **ลืมว่า Most-Derived Class ต้อง Initialize Virtual Base เอง** — ในกรณี Virtual
   Inheritance ถ้า `FlyingFish` ไม่เรียก `Animal(name)` เองใน Initializer List (ปล่อยให้
   `Flying`/`Swimming` เรียกแทน) และ `Animal` ไม่มี Default Constructor จะ Compile Error
   เพราะ Compiler พยายามเรียก `Animal()` ให้อัตโนมัติแต่หาไม่เจอ

6. **ใช้ Inheritance เพียงเพื่อ "reuse" โค้ด โดยไม่ตรวจสอบว่า is-a สมเหตุสมผลจริงหรือไม่** —
   ตัวอย่างคลาสสิกคือการสืบทอดจาก `std::vector`/`std::string`/Container มาตรฐานเพื่อยืม Method
   (ดูหัวข้อ 48.8) ซึ่งนอกจากจะละเมิดหลักการออกแบบแล้ว Container มาตรฐานของ C++ (เช่น
   `std::vector`) **ไม่มี Virtual Destructor** ด้วย ทำให้ถ้ามีใครลบ Object ผ่าน
   `std::vector<int>*` ที่ชี้ไปยัง Object ของ Class ที่สืบทอดมา (เช่น `delete
   basePtr;` โดย `basePtr` เป็น `std::vector<int>*`) จะเกิด **Undefined Behavior**
   ทันที (เรื่อง Virtual Destructor จะเรียนอย่างละเอียดใน Part 49)

7. **สับสนระหว่าง "Vehicle มี Engine" (has-a) กับ "Car เป็น Vehicle" (is-a)** — มือใหม่บางคน
   สืบทอด `Car : public Engine` เพราะคิดว่า "Car มี Engine" ซึ่งผิดหลักการโดยสิ้นเชิง ที่ถูกต้อง
   คือ `Car` ควรมี `Engine` เป็น Member Variable (Composition) เพราะ "Car is-a Engine" ฟังแล้ว
   ไม่สมเหตุสมผลเลย ในขณะที่ "Car has-a Engine" สมเหตุสมผลอย่างชัดเจน

---

## แบบฝึกหัดท้ายบท

1. ออกแบบ Class `Shape` ที่มี Member `name` (private, `std::string`) และ Constructor ที่รับชื่อ
   จากนั้นสร้าง Class `Circle : public Shape` ที่รับ `radius` เพิ่มเติม และมี Method `area()`
   ที่คำนวณพื้นที่วงกลม (ให้ `Circle` เรียก `Shape` Constructor พร้อมส่งชื่อ `"Circle"` ไปให้)

2. จากโค้ดต่อไปนี้ อธิบายว่าทำไมถึง Compile Error และแก้ไขให้ถูกต้อง:
   ```cpp
   class Base {
   public:
       Base(int x) : x_(x) {}
   private:
       int x_;
   };
   class Derived : public Base {
   public:
       Derived(int x, int y) : y_(y) {}
   private:
       int y_;
   };
   ```

3. เขียน 3-level Multilevel Inheritance ของตัวเอง (เช่น `Employee` → `Manager` → `Director`)
   โดยแต่ละระดับเพิ่ม Member Variable ของตัวเอง 1 ตัว และมี Method ที่พิมพ์ข้อมูลทั้งหมดของ
   ทุกระดับออกมาพร้อมกัน

4. ออกแบบ Class `Employee` (มี `name`, `baseSalary`) และ Class `Manager : public Employee`
   ที่เพิ่ม `bonus` และ Method `totalSalary()` ที่คำนวณ `baseSalary + bonus`

5. อธิบายด้วยคำพูดของตัวเอง (เขียนเป็น comment หรือข้อความสั้นๆ) ว่า Diamond Problem เกิดขึ้น
   ได้อย่างไร และทำไม Virtual Inheritance ถึงแก้ปัญหานี้ได้ พร้อมยกตัวอย่างสถานการณ์ในโลกจริง
   ที่อาจเจอปัญหานี้ (นอกเหนือจากตัวอย่าง Flying/Swimming ที่เรียนมา)

6. วิเคราะห์ว่าการออกแบบต่อไปนี้ควรใช้ Inheritance หรือ Composition และอธิบายเหตุผล จากนั้น
   เขียนโค้ดตามแนวทางที่เลือก:
   - `class Playlist : public std::vector<Song>` (Playlist ที่เก็บเพลง)
   - `class Car : public Engine` (รถยนต์ที่มีเครื่องยนต์)

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <string>

class Shape {
public:
    explicit Shape(std::string name) : name_(std::move(name)) {}
    const std::string& name() const { return name_; }
private:
    std::string name_;
};

class Circle : public Shape {
public:
    Circle(double radius) : Shape("Circle"), radius_(radius) {}

    double area() const { return 3.14159265358979 * radius_ * radius_; }

private:
    double radius_;
};

int main() {
    Circle c(3.0);
    std::cout << c.name() << " area=" << c.area() << '\n';
    return 0;
}
```

```
Circle area=28.2743
```

### แนวทางเฉลยข้อ 4

```cpp
#include <iostream>
#include <string>

class Employee {
public:
    Employee(std::string name, double baseSalary)
        : name_(std::move(name)), baseSalary_(baseSalary) {}

    const std::string& name() const { return name_; }
    double baseSalary() const { return baseSalary_; }

    void printSlip() const {
        std::cout << name_ << ": เงินเดือนพื้นฐาน " << baseSalary_ << " บาท\n";
    }

protected:
    std::string name_;
    double baseSalary_;
};

// Manager เพิ่ม bonus_ และ method ใหม่ totalSalary() ของตัวเอง
// (หมายเหตุ: ในบทนี้เรายังไม่ใช้ virtual function — ถ้าอยากให้ e.printSlip()/m.printSlip()
//  แสดงยอดเงินที่ถูกต้องโดยอัตโนมัติผ่าน base pointer/reference จะต้องใช้ virtual
//  ซึ่งเป็นหัวข้อหลักของ Part 49)
class Manager : public Employee {
public:
    Manager(std::string name, double baseSalary, double bonus)
        : Employee(std::move(name), baseSalary), bonus_(bonus) {}

    double totalSalary() const { return baseSalary_ + bonus_; }

    void printManagerSlip() const {
        std::cout << name_ << ": เงินเดือนรวม (พื้นฐาน+โบนัส) = " << totalSalary() << " บาท\n";
    }

private:
    double bonus_;
};

int main() {
    Employee e("Somchai", 30000.0);
    Manager m("Somsri", 50000.0, 15000.0);

    e.printSlip();          // เรียก method ที่สืบทอดมาจาก Employee
    m.printManagerSlip();   // เรียก method เฉพาะของ Manager

    return 0;
}
```

```
Somchai: เงินเดือนพื้นฐาน 30000 บาท
Somsri: เงินเดือนรวม (พื้นฐาน+โบนัส) = 65000 บาท
```

**สังเกต**: ในข้อ 4 นี้เรายังจงใจ**ไม่ใช้** `virtual` เลย เพราะ Part นี้ยังไม่ได้เรียนเรื่อง
Polymorphism — ถ้าลองเรียก `e.printSlip()` และ `m.printSlip()` (สมมติว่า `printSlip()` ถูก
Override ใน `Manager` โดยไม่มี `virtual`) ผ่าน `Employee&` ที่ชี้ไปยัง Object จริงเป็น `Manager`
จะได้ผลลัพธ์ที่ผิดพลาด (เรียก Version ของ `Employee` เสมอ ไม่ใช่ Version ของ `Manager`) — นี่คือ
ปัญหาที่แท้จริงที่ `virtual` keyword ถูกออกแบบมาแก้ไข ซึ่งเป็นหัวใจหลักของ **Part 49** ที่กำลังจะ
ถึง

(สำหรับข้อ 2, 3, 5, 6: ให้ผู้เรียนลองทำเองตามแนวทางในหัวข้อ 48.2, 48.5, 48.7 และ 48.8 ของ
Part นี้ — สำหรับข้อ 2 คำตอบคือต้องเพิ่ม `Base(x)` เข้าไปใน Initializer List ของ `Derived`
เช่น `Derived(int x, int y) : Base(x), y_(y) {}` และสำหรับข้อ 6 คำตอบคือทั้งสองกรณีควรใช้
Composition เพราะ "Playlist is-a vector of Song" และ "Car is-a Engine" ต่างก็ไม่สมเหตุสมผล
ตามธรรมชาติ ควรเป็น "Playlist has-a vector of Song" และ "Car has-a Engine" แทน)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ Single Inheritance และความสัมพันธ์แบบ **is-a** ผ่าน Syntax `class Derived : public
  Base`
- เข้าใจลำดับการเรียก Constructor (Base ก่อน Derived) และ Destructor (Derived ก่อน Base)
  พร้อมวิธีเรียก Base Constructor ผ่าน Initializer List
- เข้าใจเหตุผลที่แท้จริงของ `protected` ในบริบทของ Inheritance และข้อควรระวังในการใช้งาน
- แยกแยะความหมายของ Inheritance แบบ `public`, `protected`, `private` ได้
- เขียน Multilevel Inheritance และ Multiple Inheritance ได้อย่างถูกต้อง
- เข้าใจ **Diamond Problem** อย่างลึกซึ้งและแก้ไขด้วย **Virtual Inheritance** ได้
- เข้าใจหลักการ **Composition over Inheritance** และรู้วิธีตัดสินใจว่าเมื่อไหร่ควรใช้แบบไหน

Inheritance คือเสาหลักต้นที่สองของ OOP ที่เราเพิ่งเรียนจบไป แต่สิ่งที่ทำให้ Inheritance มีพลัง
อย่างแท้จริงคือความสามารถในการเรียก Method เวอร์ชันที่ "ถูกต้อง" ของ Object ผ่าน Base
Pointer/Reference โดยอัตโนมัติ — ซึ่งเป็นสิ่งที่เรายังไม่ได้ทำได้อย่างถูกต้องในตัวอย่างของ Part นี้
เลย (ดังที่เห็นในแบบฝึกหัดข้อ 4) ใน **Part 49** เราจะแก้ไขข้อจำกัดนี้ด้วยเสาหลักต้นที่สามของ
OOP: **Polymorphism และ Virtual Function**

**ต่อไป:** [Part 49 — Polymorphism และ Virtual Function](./part-049-polymorphism-virtual.md)
