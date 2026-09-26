# Part 49: Polymorphism และ Virtual Function (Step 385–392)

> Module D — เริ่มต้น C++ และ OOP | Part 49 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 385–392
> Part ก่อนหน้า: [Part 48 — Inheritance](./part-048-inheritance.md) | Part ถัดไป: [Part 50 — Abstract Class และ Interface](./part-050-abstract-classes.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง **Static Binding** (ผูกที่ Compile Time) กับ **Dynamic Binding**
   (ผูกที่ Runtime) และบอกได้ว่า C++ ใช้แบบไหนเป็น default
2. อธิบายได้ว่า `virtual` Function คืออะไร และแสดงให้เห็นบั๊กจริงที่เกิดขึ้นถ้าลืมใส่ `virtual`
3. อธิบายกลไกเบื้องหลังของ Virtual Function ผ่าน **vtable** (Virtual Table) และ **vptr**
   (Virtual Pointer) ได้ด้วย Diagram
4. ใช้ `override` Keyword (C++11) เพื่อป้องกันบั๊กจาก Signature ไม่ตรงกันระหว่าง Base และ
   Derived Class
5. อธิบายปัญหา **Object Slicing** และรู้วิธีป้องกัน
6. อธิบายว่าทำไม **Virtual Destructor** ถึงสำคัญมาก และสาธิต Memory Leak จริงที่เกิดขึ้นถ้าลืมใส่
7. ใช้ `final` Keyword เพื่อล็อกไม่ให้ Method หรือ Class ถูก Override/Inherit ต่อ

---

## 49.1 Static Binding vs Dynamic Binding (Step 385)

ก่อนจะเข้าใจ `virtual` เราต้องเข้าใจแนวคิดพื้นฐานก่อนว่า **"เมื่อเรียกฟังก์ชันสมาชิกของ Object
ผ่าน Base Pointer หรือ Base Reference คอมไพเลอร์ตัดสินใจอย่างไรว่าจะเรียก Version ไหน"**

มีสองแนวทาง:

| แนวทาง | ตัดสินใจตอนไหน | ใช้ข้อมูลอะไรตัดสินใจ |
|---|---|---|
| **Static Binding** (Early Binding) | Compile Time | **ชนิดที่ประกาศไว้** (Declared/Static Type) ของตัวแปร เช่น `Animal&` |
| **Dynamic Binding** (Late Binding) | Runtime | **ชนิดจริง** (Dynamic Type) ของ Object ที่ตัวแปรนั้นชี้ไปจริงๆ |

**C++ ใช้ Static Binding เป็นค่าเริ่มต้นสำหรับ Member Function ทุกตัว** ยกเว้นฟังก์ชันที่ประกาศ
เป็น `virtual` อย่างชัดเจน ซึ่งจะใช้ Dynamic Binding แทน — นี่คือสิ่งที่มือใหม่แทบทุกคนพลาดตอนเริ่ม
เรียน Inheritance เพราะคิดว่า "แค่มี Method ชื่อเดียวกันใน Derived Class ก็คือ Override แล้ว"
ซึ่ง **ไม่จริง**

### สาธิตปัญหาจริง: ลืมใส่ virtual

```cpp
#include <iostream>
#include <string>

// *** ตัวอย่างนี้จงใจแสดงบั๊ก: ลืมใส่ virtual ***
// โค้ดนี้คอมไพล์ผ่านโดยไม่มี Warning ใดๆ เลย (แม้เปิด -Wall -Wextra -Wpedantic)
// แต่ผลลัพธ์ตอนรันกลับผิดพลาด เพราะ speak() ไม่ใช่ virtual function

class Animal {
public:
    explicit Animal(std::string name) : name_(std::move(name)) {}

    void speak() const {   // ไม่มี virtual -> resolve ตอน compile time (Static Binding)
        std::cout << name_ << " (Animal): ...\n";
    }

protected:
    std::string name_;
};

class Dog : public Animal {
public:
    explicit Dog(std::string name) : Animal(std::move(name)) {}

    void speak() const {   // ตั้งใจ "override" แต่จริงๆ แค่ hide ฟังก์ชันของ base เฉยๆ
        std::cout << name_ << " (Dog): โฮ่ง โฮ่ง!\n";
    }
};

void makeItSpeak(const Animal& a) {
    a.speak();   // เรียกผ่าน reference ที่ประกาศเป็น Animal&
}

int main() {
    Dog d("โปงลาง");

    d.speak();            // เรียกตรงๆ ผ่าน Dog -> ได้ผลถูกต้อง "โฮ่ง โฮ่ง!"
    makeItSpeak(d);        // เรียกผ่าน Animal& -> ควรได้ "โฮ่ง โฮ่ง!" เหมือนกัน แต่กลับไม่ใช่!

    return 0;
}
```

```
โปงลาง (Dog): โฮ่ง โฮ่ง!
โปงลาง (Animal): ...
```

**นี่คือบั๊กที่อันตรายที่สุดแบบหนึ่งใน C++ เพราะมันไม่มี Warning ใดๆ เลยแม้เปิด `-Wall -Wextra
-Wpedantic` ครบถ้วน** โค้ดคอมไพล์ผ่านสนิท แต่ผลลัพธ์ผิดจากที่คาดหวังโดยสิ้นเชิง

**อธิบายว่าทำไม**: เมื่อ `makeItSpeak` รับพารามิเตอร์เป็น `const Animal&` การเรียก `a.speak()`
ภายในฟังก์ชันนี้ **ถูกผูกไว้ตอน Compile Time** ตามชนิดที่ **ประกาศไว้** ของ `a` ซึ่งคือ
`Animal` เสมอ ไม่ว่า Object จริงที่ `a` อ้างถึงจะเป็น `Dog`, `Cat`, หรือ Class ลูกอื่นใดก็ตาม
Compiler ไม่สนใจ "ชนิดจริง" ของ Object เลยถ้าฟังก์ชันนั้นไม่ใช่ `virtual`

---

## 49.2 virtual Function: ทางแก้ปัญหา Dynamic Binding (Step 386)

การเติมคำว่า `virtual` หน้า Member Function บอกให้ Compiler รู้ว่า **"ฟังก์ชันนี้ต้องถูกเลือก
Version ที่ถูกต้องตามชนิดจริงของ Object ตอน Runtime เสมอ"** — นี่คือหัวใจของ **Polymorphism**
(ความสามารถของ Object หลายชนิดที่มีความสัมพันธ์แบบ Inheritance กัน ให้ "ตอบสนอง" ต่อการเรียก
Method เดียวกันด้วยพฤติกรรมที่แตกต่างกันไปตามชนิดจริงของแต่ละตัว)

```cpp
#include <iostream>
#include <string>

class Animal {
public:
    explicit Animal(std::string name) : name_(std::move(name)) {}

    virtual void speak() const {   // มี virtual -> resolve ตอน runtime (Dynamic Binding)
        std::cout << name_ << " (Animal): ...\n";
    }

    virtual ~Animal() = default;   // จะอธิบายเหตุผลอย่างละเอียดในหัวข้อ 49.6

protected:
    std::string name_;
};

class Dog : public Animal {
public:
    explicit Dog(std::string name) : Animal(std::move(name)) {}

    void speak() const override {   // override ของจริง เพราะ base เป็น virtual
        std::cout << name_ << " (Dog): โฮ่ง โฮ่ง!\n";
    }
};

void makeItSpeak(const Animal& a) {
    a.speak();   // ตอนนี้จะเรียก version ที่ถูกต้องตาม "ชนิดจริง" ของ object เสมอ
}

int main() {
    Dog d("โปงลาง");

    d.speak();
    makeItSpeak(d);   // ตอนนี้ถูกต้องแล้ว: เรียก Dog::speak() ผ่าน Animal& ได้อย่างถูกต้อง

    return 0;
}
```

```
โปงลาง (Dog): โฮ่ง โฮ่ง!
โปงลาง (Dog): โฮ่ง โฮ่ง!
```

ตอนนี้ผลลัพธ์ถูกต้องทั้งสองกรณี เพราะ `speak()` ถูกทำเครื่องหมายเป็น `virtual` ใน `Animal`
ทำให้ Compiler สร้างกลไกพิเศษ (จะอธิบายในหัวข้อถัดไป) ที่คอย "มองหา" Version ที่แท้จริงของ
`speak()` จากชนิดจริงของ Object ทุกครั้งที่ถูกเรียกผ่าน Pointer/Reference ของ Base Class

> **กฎทองข้อใหม่ของหลักสูตรนี้**: ถ้า Class ใดถูกออกแบบมาให้เป็น Base Class ของ Inheritance
> (คือคาดว่าจะมี Class อื่นสืบทอดไป) และมี Method ที่ต้องการให้ Derived Class ปรับพฤติกรรมได้
> ให้ประกาศ Method นั้นเป็น `virtual` เสมอ

---

## 49.3 กลไกเบื้องหลัง: vtable และ vptr (Step 387)

Dynamic Binding ไม่ใช่เวทมนตร์ — มันมีกลไกที่ชัดเจนอยู่เบื้องหลังที่เรียกว่า **Virtual Table
(vtable)** และ **Virtual Pointer (vptr)** เข้าใจกลไกนี้จะช่วยให้เราเข้าใจว่าทำไม Virtual
Function ถึงมี Overhead เล็กน้อย (แต่คุ้มค่ามากเมื่อเทียบกับความยืดหยุ่นที่ได้มา) และทำไม
Object ที่มี Virtual Function ถึงมีขนาดใหญ่ขึ้นกว่าปกติเล็กน้อย

### แนวคิดหลัก

เมื่อ Class ใดมี Virtual Function อย่างน้อย 1 ตัว Compiler จะทำสิ่งต่อไปนี้โดยอัตโนมัติ:

1. สร้างตารางที่เรียกว่า **vtable** ขึ้นมา **หนึ่งตารางต่อหนึ่ง Class** (ไม่ใช่ต่อ Object)
   ตารางนี้เก็บ **Address ของ Virtual Function เวอร์ชันที่ถูกต้องของ Class นั้น** ไว้เป็นแถวๆ
2. Object ทุกตัวของ Class ที่มี Virtual Function จะมี **Hidden Pointer** ซ่อนอยู่ (มองไม่เห็น
   ในโค้ด) เรียกว่า **vptr** ที่ชี้ไปยัง vtable ของ Class ที่ตัวเองเป็นชนิดจริง

```
vtable ของ Animal                    vtable ของ Dog
┌─────────────────────┐              ┌─────────────────────┐
│ speak → Animal::speak│              │ speak → Dog::speak   │
│ ~Animal → Animal::~..│              │ ~Dog → Dog::~..       │
└─────────────────────┘              └─────────────────────┘
        ▲                                      ▲
        │                                      │
   ┌────┴─────┐                          ┌─────┴────┐
   │  vptr    │  <- Object ของ Animal    │  vptr    │  <- Object ของ Dog
   │ name_    │                          │ name_    │
   └──────────┘                          └──────────┘
   Animal object                         Dog object
```

เมื่อเราเรียก `a.speak()` โดยที่ `a` เป็น `Animal&` (Static Type คือ `Animal`) แต่ Object จริง
เป็น `Dog` (Dynamic Type คือ `Dog`) กระบวนการทำงานคือ:

```
1. เข้าไปดู vptr ของ object (ซึ่งชี้ไปยัง vtable ของ Dog เพราะ object จริงเป็น Dog)
2. เปิด vtable ของ Dog แล้วมองหาแถวของ speak()
3. พบ address ของ Dog::speak() -> เรียกฟังก์ชันนั้น
```

ทั้งหมดนี้เกิดขึ้น **ตอน Runtime** (ทุกครั้งที่เรียก Virtual Function) จึงเรียกว่า Dynamic
Binding — และเป็นเหตุผลว่าทำไมการเรียก Virtual Function จึงมี Overhead เล็กน้อยกว่าการเรียก
ฟังก์ชันธรรมดา (ต้องเดินทางผ่าน vptr → vtable → address จริง แทนที่จะกระโดดไปยัง address ที่
รู้แน่นอนอยู่แล้วตอน Compile Time) แต่ในทางปฏิบัติ Overhead นี้เล็กน้อยมากเมื่อเทียบกับ
ความยืดหยุ่นในการออกแบบที่ได้มา

### ทำไม sizeof(Object) ถึงใหญ่ขึ้นเมื่อมี virtual function

```cpp
#include <iostream>

class NoVirtual {
    int x_ = 0;
};

class HasVirtual {
    int x_ = 0;
public:
    virtual ~HasVirtual() = default;
};

int main() {
    std::cout << "sizeof(NoVirtual)  = " << sizeof(NoVirtual) << " bytes\n";
    std::cout << "sizeof(HasVirtual) = " << sizeof(HasVirtual) << " bytes\n";
    return 0;
}
```

```
sizeof(NoVirtual)  = 4 bytes
sizeof(HasVirtual) = 16 bytes
```

`HasVirtual` ใหญ่ขึ้นเพราะต้องมีที่เก็บ **vptr** (ปกติมีขนาดเท่ากับ Pointer หนึ่งตัว คือ 8 bytes
บนระบบ 64-bit) บวกกับการจัด Memory Alignment ทำให้ขนาดจริงที่เห็นอาจไม่ใช่แค่ 4+8=12 bytes
ตรงๆ (ผลลัพธ์ที่แน่นอนขึ้นกับ Compiler และแพลตฟอร์ม) — ประเด็นสำคัญคือ **การมี Virtual
Function แม้แค่ตัวเดียวก็ทำให้ Object ทุกตัวของ Class นั้นมีขนาดใหญ่ขึ้น** เพราะต้องเก็บ vptr
ไว้เสมอ (แต่ vtable เองมีแค่ชุดเดียวต่อ Class ไม่ได้ซ้ำในทุก Object)

---

## 49.4 override Keyword (C++11): ป้องกันบั๊กจาก Signature ไม่ตรงกัน (Step 388)

ก่อน C++11 การ Override Virtual Function ทำได้แค่ **เขียน Signature ให้เหมือน Base Class
เป๊ะๆ ด้วยมือ** ถ้าพิมพ์ชื่อผิด หรือ const ไม่ตรง หรือ Parameter Type ไม่ตรง Compiler จะไม่เตือน
อะไรเลย เพราะมันจะกลายเป็นการสร้าง **ฟังก์ชันใหม่ที่ไม่เกี่ยวข้องกัน** (Hiding แทนที่จะเป็น
Overriding) ทันที

### สาธิตปัญหาจริง: พิมพ์ชื่อฟังก์ชันผิดโดยไม่มี override

```cpp
#include <iostream>
#include <string>

// *** ตัวอย่างนี้จงใจแสดงบั๊ก: พิมพ์ชื่อฟังก์ชันผิด แต่ไม่มี override ช่วยเตือน ***
class Animal {
public:
    explicit Animal(std::string name) : name_(std::move(name)) {}
    virtual ~Animal() = default;

    virtual void makeSound() const {
        std::cout << name_ << ": ...\n";
    }

protected:
    std::string name_;
};

class Cat : public Animal {
public:
    explicit Cat(std::string name) : Animal(std::move(name)) {}

    // พิมพ์ผิดเป็น "makeSond" โดยไม่มี override กำกับ
    // Compiler มองว่านี่คือฟังก์ชันใหม่ที่ไม่เกี่ยวกับ Animal::makeSound() เลย
    // ผลคือ "override" ที่ตั้งใจไว้ไม่เกิดขึ้นจริง แต่คอมไพล์ผ่านสนิท ไม่มี Warning เลย!
    void makeSond() const {
        std::cout << name_ << ": เมี้ยว!\n";
    }
};

int main() {
    Cat c("มะลิ");
    const Animal& a = c;

    a.makeSound();   // คาดหวังว่าจะได้ "เมี้ยว!" แต่กลับได้ "..." เพราะ override ไม่สำเร็จจริง

    return 0;
}
```

```
มะลิ: ...
```

บั๊กนี้ **หาเจอยากมาก** ในโค้ดจริง เพราะไม่มี Error หรือ Warning ใดๆ เลย และผลลัพธ์ก็ยังคอมไพล์
และรันได้ตามปกติ เพียงแต่ผลลัพธ์ผิดจากที่ตั้งใจไว้อย่างเงียบๆ

### แก้ด้วย override

`override` เป็น Keyword ที่เพิ่มเข้ามาใน C++11 ใช้เขียนต่อท้ายฟังก์ชันที่ **ตั้งใจ Override**
Virtual Function จาก Base Class โดย Compiler จะ**ตรวจสอบทันที**ว่ามี Virtual Function ใน
Base Class ที่ Signature ตรงกันเป๊ะจริงหรือไม่ ถ้าไม่พบ จะ **Compile Error ทันที**:

```cpp
#include <iostream>
#include <string>

class Animal {
public:
    explicit Animal(std::string name) : name_(std::move(name)) {}
    virtual ~Animal() = default;

    virtual void makeSound() const {
        std::cout << name_ << ": ...\n";
    }

protected:
    std::string name_;
};

class Cat : public Animal {
public:
    explicit Cat(std::string name) : Animal(std::move(name)) {}

    // พิมพ์ผิด: "makeSond" แทนที่จะเป็น "makeSound" — สัญญาณ (signature) ไม่ตรงกับ base เลย
    // ถ้าไม่มี override ตรงนี้ Compiler จะไม่รู้เลยว่าเราตั้งใจจะ override แต่ไม่สำเร็จ
    // มันจะกลายเป็นฟังก์ชันใหม่เฉยๆ ใน Cat ที่ไม่เกี่ยวอะไรกับ Animal::makeSound() เลย
    void makeSond() const override {
        std::cout << name_ << ": เมี้ยว!\n";
    }
};

int main() {
    Cat c("มะลิ");
    c.makeSond();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 override_catches_typo.cpp -o override_catches_typo
```

```
override_catches_typo.cpp:24:10: error: 'void Cat::makeSond() const' marked 'override', but does not override
   24 |     void makeSond() const override {
      |          ^~~~~~~~
```

**Compiler จับบั๊กนี้ได้ทันทีตอน Compile Time** แทนที่จะปล่อยให้เป็นบั๊กเงียบๆ ที่ต้องไล่จับตอน
Runtime — นี่คือเหตุผลที่ Modern C++ Best Practice (จะเรียนเจาะลึกใน Part 80) แนะนำอย่างหนักแน่น
ว่า:

> **ทุกครั้งที่ Override Virtual Function ให้ใส่ `override` เสมอ ไม่มีข้อยกเว้น**

`override` ไม่ได้เปลี่ยนพฤติกรรมของโปรแกรมเลยแม้แต่นิดเดียวเมื่อเขียนถูกต้อง มันเป็นแค่
**คำสัญญาที่ Compiler ช่วยตรวจสอบให้** ทำให้ไม่มีเหตุผลใดเลยที่จะไม่ใส่มันทุกครั้ง

---

## 49.5 ปัญหา Object Slicing (Step 389)

**Object Slicing** คือปัญหาที่เกิดขึ้นเมื่อเรา **Copy Object ของ Derived Class ไปเก็บใน
ตัวแปรหรือ Parameter ที่มีชนิดเป็น Base Class โดยตรง (ไม่ใช่ Pointer/Reference)** ผลคือส่วน
ของ Derived ที่เพิ่มเข้ามาจะถูก "ตัดทิ้ง" (sliced off) ไปหมด เหลือแต่ส่วนของ Base เท่านั้น

```cpp
#include <iostream>
#include <string>

class Animal {
public:
    explicit Animal(std::string name) : name_(std::move(name)) {}
    virtual ~Animal() = default;

    virtual void speak() const {
        std::cout << name_ << " (Animal): ...\n";
    }

protected:
    std::string name_;
};

class Dog : public Animal {
public:
    Dog(std::string name, std::string breed)
        : Animal(std::move(name)), breed_(std::move(breed)) {}

    void speak() const override {
        std::cout << name_ << " (" << breed_ << "): โฮ่ง โฮ่ง!\n";
    }

private:
    std::string breed_;
};

// *** ตัวอย่างนี้จงใจแสดงบั๊ก: Object Slicing ***
// รับพารามิเตอร์เป็น "Animal ตรงๆ" (by value) ไม่ใช่ Animal& หรือ Animal*
void describeSliced(Animal a) {   // a ถูกสร้างเป็น Animal ใหม่ ก็อปปี้แค่ส่วนของ Animal เท่านั้น
    a.speak();   // เรียก Animal::speak() เสมอ เพราะ a เป็น Animal "แท้ๆ" ไปแล้ว ไม่ใช่ Dog อีกต่อไป
}

void describeCorrect(const Animal& a) {   // รับเป็น reference -> ไม่มีการก็อปปี้ ไม่มี slicing
    a.speak();
}

int main() {
    Dog d("โปงลาง", "Golden Retriever");

    std::cout << "เรียกผ่าน reference (ถูกต้อง):\n";
    describeCorrect(d);

    std::cout << "เรียกผ่าน by-value (เกิด object slicing):\n";
    describeSliced(d);   // d ถูก "ตัด" ส่วนของ Dog ทิ้งไปหมด เหลือแค่ส่วนของ Animal

    return 0;
}
```

```
เรียกผ่าน reference (ถูกต้อง):
โปงลาง (Golden Retriever): โฮ่ง โฮ่ง!
เรียกผ่าน by-value (เกิด object slicing):
โปงลาง (Animal): ...
```

**อธิบายว่าทำไม**: เมื่อ `describeSliced(Animal a)` รับพารามิเตอร์เป็น `Animal` **by value**
Compiler จะสร้าง Object ของ `Animal` ขึ้นมาใหม่โดยใช้ **Copy Constructor ของ `Animal`
เท่านั้น** (ไม่ใช่ของ `Dog`) เพราะ `a` ถูกประกาศเป็นชนิด `Animal` ตรงๆ ผลคือ `breed_` (ที่มี
แค่ใน `Dog`) หายไปโดยสิ้นเชิง และแม้ `speak()` จะเป็น `virtual` ก็ไม่ช่วยอะไรเลย เพราะ vptr
ของ `a` ถูกตั้งค่าใหม่ให้ชี้ไปยัง vtable ของ `Animal` ตั้งแต่ตอน Copy แล้ว (Virtual Function
แก้ปัญหาเรื่อง Dynamic Binding ผ่าน **Pointer/Reference** เท่านั้น ไม่ได้แก้ปัญหาเรื่องการ
**Copy Object โดยตรง**)

### วิธีป้องกัน Object Slicing

| วิธี | อธิบาย |
|---|---|
| **รับพารามิเตอร์เป็น Reference หรือ Pointer เสมอ** เมื่อต้องการ Polymorphism | `const Animal&` หรือ `const Animal*` แทน `Animal` ตรงๆ |
| **เก็บ Object แบบ Polymorphic ใน Container ด้วย Pointer** เสมอ | เช่น `std::vector<std::unique_ptr<Animal>>` แทน `std::vector<Animal>` |
| **ห้าม Copy Object ของ Class ที่ออกแบบมาเพื่อ Polymorphism** ถ้าเป็นไปได้ | บาง Codebase จะ `delete` Copy Constructor ของ Base Class ที่มี Virtual Function โดยตั้งใจ เพื่อบังคับให้ใช้ Pointer/Reference เท่านั้น |

> **กฎทองสำหรับ Part นี้**: เมื่อออกแบบ Class ที่จะใช้ในลักษณะ Polymorphism (มี Virtual
> Function และคาดว่าจะมี Class ลูกหลายตัว) ให้ **ส่งผ่าน Function Parameter เป็น Reference
> หรือ Pointer เท่านั้น** ห้ามส่งเป็น Value โดยเด็ดขาด

---

## 49.6 Virtual Destructor: ทำไมสำคัญมาก (Step 390)

นี่คือหัวข้อที่สำคัญที่สุดหัวข้อหนึ่งของทั้ง Module D **การลืมใส่ `virtual` ให้ Destructor ของ
Base Class ที่ถูกออกแบบมาให้มี Class ลูก คือสาเหตุของ Memory Leak ที่พบบ่อยที่สุดอันดับต้นๆ
ในโค้ด C++ ที่เขียนโดยมือใหม่**

### สาธิตปัญหาจริง: Memory Leak จากการลืม virtual destructor

```cpp
#include <iostream>

// *** ตัวอย่างนี้จงใจแสดงบั๊ก: ไม่มี virtual destructor ***
class Base {
public:
    Base() { std::cout << "Base constructor\n"; }
    ~Base() { std::cout << "~Base destructor\n"; }   // ไม่ใช่ virtual!
};

class Derived : public Base {
public:
    Derived() {
        data_ = new int[100];   // จำลองทรัพยากรที่ Derived จองไว้เอง
        std::cout << "Derived constructor (จอง memory ไว้)\n";
    }
    ~Derived() {
        delete[] data_;
        std::cout << "~Derived destructor (คืน memory แล้ว)\n";
    }

private:
    int* data_;
};

int main() {
    std::cout << "--- ลบผ่าน Derived* โดยตรง (ไม่มีปัญหา) ---\n";
    Derived* d1 = new Derived();
    delete d1;   // เรียก ~Derived() แล้วตามด้วย ~Base() ถูกต้องครบ

    std::cout << "\n--- ลบผ่าน Base* ที่ชี้ไปยัง Derived (มีปัญหา!) ---\n";
    Base* b = new Derived();   // Base* ชี้ไปยัง Derived object จริง (Object นี้ is-a Base ด้วย)
    delete b;   // เพราะ ~Base() ไม่ใช่ virtual -> เรียกแค่ ~Base() เท่านั้น!
                // ~Derived() "ไม่ถูกเรียกเลย" -> data_ (int[100]) รั่วไหล (memory leak) ถาวร

    std::cout << "\nจบโปรแกรม (สังเกตว่าไม่มีข้อความ \"~Derived destructor\" ครั้งที่ 2 เลย)\n";
    return 0;
}
```

```
--- ลบผ่าน Derived* โดยตรง (ไม่มีปัญหา) ---
Base constructor
Derived constructor (จอง memory ไว้)
~Derived destructor (คืน memory แล้ว)
~Base destructor

--- ลบผ่าน Base* ที่ชี้ไปยัง Derived (มีปัญหา!) ---
Base constructor
Derived constructor (จอง memory ไว้)
~Base destructor

จบโปรแกรม (สังเกตว่าไม่มีข้อความ "~Derived destructor" ครั้งที่ 2 เลย)
```

สังเกตให้ดี: ในการลบครั้งที่สอง **ไม่มีข้อความ `"~Derived destructor"` ปรากฏขึ้นเลย** ซึ่งหมายความ
ว่า `delete[] data_;` **ไม่เคยถูกเรียกเลย** — Memory 100 ตัว `int` ที่ `Derived` จองไว้ **รั่วไหล
ถาวร** ไปตลอดอายุของโปรแกรม (ถ้าลองรันโปรแกรมนี้ผ่าน Valgrind ที่เรียนใน Part 38 จะเห็น
"definitely lost" block ปรากฏขึ้นทันที) และที่อันตรายยิ่งกว่าคือ **โค้ดนี้คอมไพล์ผ่านสนิทโดย
ไม่มี Warning ใดๆ เลย** แม้เปิด `-Wall -Wextra -Wpedantic` ครบถ้วน เพราะ `Base` ในตัวอย่างนี้
ไม่มี Virtual Function ตัวอื่นเลยนอกจาก Destructor Compiler จึงไม่มีเบาะแสว่า Class นี้จะถูกใช้
งานแบบ Polymorphism

### แก้ด้วย virtual destructor

```cpp
#include <iostream>

class Base {
public:
    Base() { std::cout << "Base constructor\n"; }
    virtual ~Base() { std::cout << "~Base destructor\n"; }   // เพิ่ม virtual แล้ว
};

class Derived : public Base {
public:
    Derived() {
        data_ = new int[100];
        std::cout << "Derived constructor (จอง memory ไว้)\n";
    }
    ~Derived() override {
        delete[] data_;
        std::cout << "~Derived destructor (คืน memory แล้ว)\n";
    }

private:
    int* data_;
};

int main() {
    std::cout << "--- ลบผ่าน Base* ที่ชี้ไปยัง Derived (ตอนนี้ถูกต้องแล้ว) ---\n";
    Base* b = new Derived();
    delete b;   // ตอนนี้ ~Base() เป็น virtual -> เรียก ~Derived() ก่อน แล้วค่อยเรียก ~Base()
                // ตามลำดับที่ถูกต้อง ไม่มี memory leak อีกต่อไป

    return 0;
}
```

```
--- ลบผ่าน Base* ที่ชี้ไปยัง Derived (ตอนนี้ถูกต้องแล้ว) ---
Base constructor
Derived constructor (จอง memory ไว้)
~Derived destructor (คืน memory แล้ว)
~Base destructor
```

**กลไกเบื้องหลัง**: เมื่อ Destructor เป็น `virtual` การเรียก `delete b;` จะทำงานผ่าน vtable
เหมือนกับ Virtual Function ทั่วไป — มันจะมองหา Destructor ที่ถูกต้องตามชนิดจริงของ Object
(`Derived`) เจอ `~Derived()` ก่อน เรียกมันก่อน แล้ว `~Derived()` เมื่อจบตัวเองจะเรียก
`~Base()` ต่อให้อัตโนมัติเสมอ (เหมือนที่เรียนเรื่องลำดับ Destructor ใน Part 48)

> **กฎทองที่สำคัญที่สุดข้อหนึ่งของทั้งหลักสูตร**:
> **ถ้า Class ใดมี Virtual Function อย่างน้อยหนึ่งตัว (หรือถูกออกแบบให้เป็น Base Class ของ
> Inheritance ที่จะถูกลบผ่าน Base Pointer) ให้ Destructor ของ Class นั้นเป็น `virtual` เสมอ
> ไม่มีข้อยกเว้น** แม้ Destructor นั้นจะไม่มีอะไรทำเลยก็ตาม (`virtual ~Base() = default;`)

Compiler บางตัวมี Flag พิเศษ (`-Wnon-virtual-dtor` ซึ่งไม่ได้รวมอยู่ใน `-Wall`/`-Wextra`
โดยตรง) ที่ช่วยเตือนกรณี Class มี Virtual Function อื่นแต่ Destructor ไม่ใช่ Virtual — แต่ไม่ได้
เตือนในทุกกรณี (ดังตัวอย่างข้างต้นที่ `Base` ไม่มี Virtual Function อื่นเลย) ดังนั้น **อย่าพึ่งพา
Compiler Warning เพียงอย่างเดียว ต้องจำกฎทองข้อนี้ไว้ในใจเสมอ**

---

## 49.7 final Keyword (Step 391)

`final` เป็นอีก Keyword ที่เพิ่มมาใน C++11 ใช้ **ล็อก** ไม่ให้มีการ Override หรือ Inherit
ต่อไปอีก มีสองรูปแบบการใช้งาน:

### final กับ Method: ห้าม Override ต่อ

```cpp
virtual bool validate() const final { /* ... */ }
```

### final กับ Class: ห้าม Inherit ต่อ

```cpp
class Square final : public Shape { /* ... */ };
```

```cpp
#include <iostream>
#include <string>

class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const { return 0.0; }   // ค่าเริ่มต้น: class ลูกควร override เสมอ

    // validate() ถูกล็อกด้วย final: ห้าม class ลูกใดๆ override อีกต่อไป
    // เหมาะกับ method ที่มี logic สำคัญที่ไม่ควรให้ class ลูกเปลี่ยนแปลงพฤติกรรมได้
    virtual bool validate() const final {
        return area() > 0.0;
    }
};

class Square final : public Shape {   // final ที่ class: ห้ามมีใคร inherit จาก Square อีก
public:
    explicit Square(double side) : side_(side) {}
    double area() const override { return side_ * side_; }

private:
    double side_;
};

// class Hacked : public Square {};   // ผิด: Square เป็น final class ห้ามสืบทอดต่อ

int main() {
    Square sq(4.0);
    std::cout << "area=" << sq.area() << " valid=" << std::boolalpha << sq.validate() << '\n';
    return 0;
}
```

```
area=16 valid=true
```

ถ้าลองมี Class ที่พยายาม Override Method ที่เป็น `final`:

```cpp
class Shape {
public:
    virtual ~Shape() = default;
    virtual bool validate() const final { return true; }
};

class Square : public Shape {
public:
    bool validate() const override { return false; }   // ผิด: validate() เป็น final ใน Shape แล้ว
};

int main() {
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 final_violation.cpp -o final_violation
```

```
final_violation.cpp:9:10: error: virtual function 'virtual bool Square::validate() const' overriding final function
    9 |     bool validate() const override { return false; }
      |          ^~~~~~~~
final_violation.cpp:4:18: note: overridden function is 'virtual bool Shape::validate() const'
```

**เมื่อไหร่ควรใช้ `final`**:

- ใช้กับ **Method** เมื่อมี Logic สำคัญที่ไม่ควรให้ Class ลูกใดๆ เปลี่ยนแปลงพฤติกรรมได้เลย
  (เช่น Method ที่ตรวจสอบ Invariant สำคัญ)
- ใช้กับ **Class** เมื่อ Class นั้นถูกออกแบบมาให้เป็น "จุดสิ้นสุด" ของสายโซ่ Inheritance
  จริงๆ (เช่น Class ที่ Optimize มาอย่างละเอียดแล้ว และการสืบทอดต่อไปอาจทำลาย Invariant หรือ
  Performance ที่ตั้งใจไว้)
- เป็นเครื่องมือสื่อสารเจตนาที่ชัดเจนให้กับคนอ่านโค้ดคนอื่น (รวมถึงตัวเราเองในอนาคต) ว่า
  "ตรงนี้ตั้งใจให้จบแค่นี้ ห้ามขยายต่อ"

---

## 49.8 ตัวอย่างสรุปรวม: Polymorphism ที่สมบูรณ์แบบ (Step 392)

มารวมทุกหลักการที่เรียนมาใน Part นี้: `virtual` + `override` + Virtual Destructor + การเก็บ
Object หลายชนิดผ่าน Base Pointer ใน Container เดียวกัน (การใช้งาน Polymorphism ที่พบบ่อยที่สุด
ในโค้ดจริง)

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>

/*
 * ตัวอย่างสรุปรวมของ Polymorphism และ Virtual Function
 * - virtual function + override สำหรับ dynamic dispatch ที่ถูกต้อง
 * - virtual destructor เพื่อป้องกัน memory leak
 * - เก็บ object หลายชนิดผ่าน base pointer ใน container เดียว (classic polymorphism use case)
 */
class Shape {
public:
    explicit Shape(std::string name) : name_(std::move(name)) {}
    virtual ~Shape() { std::cout << "  ~Shape(" << name_ << ")\n"; }

    virtual double area() const { return 0.0; }   // ค่าเริ่มต้น: class ลูกควร override เสมอ

    void printInfo() const {
        // เรียก area() ซึ่งเป็น virtual -> dynamic dispatch เลือก version ที่ถูกต้องเสมอ
        std::cout << name_ << ": area = " << area() << '\n';
    }

protected:
    std::string name_;
};

class Circle final : public Shape {   // final: ห้ามมีใครสืบทอดจาก Circle ต่อ
public:
    Circle(std::string name, double radius) : Shape(std::move(name)), radius_(radius) {}
    ~Circle() override { std::cout << "  ~Circle(" << name_ << ")\n"; }

    double area() const override { return 3.14159265358979 * radius_ * radius_; }

private:
    double radius_;
};

class Rectangle final : public Shape {
public:
    Rectangle(std::string name, double w, double h)
        : Shape(std::move(name)), width_(w), height_(h) {}
    ~Rectangle() override { std::cout << "  ~Rectangle(" << name_ << ")\n"; }

    double area() const override { return width_ * height_; }

private:
    double width_;
    double height_;
};

int main() {
    // เก็บ Shape หลายชนิดผ่าน unique_ptr<Shape> (base pointer) ใน vector เดียวกัน
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>("วงกลม A", 3.0));
    shapes.push_back(std::make_unique<Rectangle>("สี่เหลี่ยม B", 4.0, 5.0));

    std::cout << "แสดงพื้นที่ของแต่ละรูป (dynamic dispatch เลือก area() ที่ถูกต้องอัตโนมัติ):\n";
    for (const auto& shape : shapes) {
        shape->printInfo();
    }

    std::cout << "จบโปรแกรม (vector ถูกทำลาย -> destructor แต่ละรูปถูกเรียกอย่างถูกต้อง):\n";
    return 0;
}
```

```
แสดงพื้นที่ของแต่ละรูป (dynamic dispatch เลือก area() ที่ถูกต้องอัตโนมัติ):
วงกลม A: area = 28.2743
สี่เหลี่ยม B: area = 20
จบโปรแกรม (vector ถูกทำลาย -> destructor แต่ละรูปถูกเรียกอย่างถูกต้อง):
  ~Circle(วงกลม A)
  ~Shape(วงกลม A)
  ~Rectangle(สี่เหลี่ยม B)
  ~Shape(สี่เหลี่ยม B)
```

สังเกตจุดสำคัญ: `std::vector<std::unique_ptr<Shape>>` เก็บ Object ที่มีชนิดจริงต่างกัน
(`Circle`, `Rectangle`) ผ่าน `Shape*` (โดยห่อด้วย `std::unique_ptr` ซึ่งเป็น Smart Pointer
จะเรียนอย่างละเอียดใน Part 67) ในตัวแปรเดียวกันได้อย่างเป็นธรรมชาติ และเมื่อเรียก
`shape->printInfo()` ซึ่งเรียก `area()` ที่เป็น `virtual` ภายใน ผลลัพธ์จะถูกต้องตามชนิดจริง
ของแต่ละ Object เสมอ นี่คือพลังที่แท้จริงของ Polymorphism: **เขียน Loop เดียว ทำงานถูกต้องกับ
ทุกชนิดของ Shape ที่มีอยู่ในปัจจุบันและที่จะถูกเพิ่มเข้ามาในอนาคต โดยไม่ต้องแก้โค้ด Loop เลย**
(หลักการนี้เรียกว่า **Open-Closed Principle** จะเรียนอย่างละเอียดใน Part 80)

และเพราะ `~Shape()` เป็น `virtual` การทำลาย `unique_ptr` แต่ละตัวใน `vector` (เกิดขึ้น
อัตโนมัติเมื่อ `vector` ออกจาก Scope) จึงเรียก Destructor ของชนิดจริงก่อนเสมอ (`~Circle()`,
`~Rectangle()`) แล้วตามด้วย `~Shape()` อย่างถูกต้อง ไม่มี Memory Leak

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมใส่ `virtual` ให้ Method ที่ควรถูก Override** — ทำให้เกิด Function Hiding แทน Function
   Overriding ผลลัพธ์คือเรียก Method ผิด Version เมื่อเรียกผ่าน Base Pointer/Reference
   (ดูตัวอย่างในหัวข้อ 49.1) **นี่คือบั๊กที่อันตรายที่สุด เพราะไม่มี Compiler Warning เตือนเลย**

2. **ลืมใส่ `virtual` ให้ Destructor ของ Base Class** — ทำให้ Destructor ของ Derived Class
   ไม่ถูกเรียกเมื่อลบ Object ผ่าน Base Pointer เกิด **Memory Leak** หรือแย่กว่านั้นคือ
   **Undefined Behavior** ถ้า Derived Class ถือ Resource ที่ต้องปล่อยคืน (ไฟล์, Socket, Lock)
   นี่คือกฎทองที่สำคัญที่สุดข้อหนึ่งของ Part นี้ (ดูหัวข้อ 49.6)

3. **ไม่ใส่ `override` เมื่อ Override Virtual Function** — ทำให้พลาดจับบั๊กจาก Signature
   ไม่ตรงกัน (พิมพ์ชื่อผิด, const ไม่ตรง, Parameter Type ไม่ตรง) ตั้งแต่ตอน Compile Time
   ทั้งที่ Compiler สามารถช่วยจับให้ได้ฟรีๆ ถ้าใส่ `override` ไว้ (ดูหัวข้อ 49.4)

4. **Object Slicing จากการส่ง Parameter หรือเก็บ Object แบบ Polymorphic เป็น Value** —
   ทำให้ส่วนของ Derived Class หายไปหมด เหลือแค่ส่วนของ Base เท่านั้น วิธีป้องกันคือใช้
   Reference หรือ Pointer เสมอเมื่อทำงานกับ Object แบบ Polymorphism (ดูหัวข้อ 49.5)

5. **สับสนระหว่าง Overriding กับ Overloading** — Overriding คือการเขียน Signature เดียวกัน
   เป๊ะใน Derived Class เพื่อเปลี่ยนพฤติกรรมของ Virtual Function (ต้องมี `virtual` ใน Base)
   ส่วน Overloading (เรียนใน Part 44) คือการมีฟังก์ชันชื่อเดียวกันแต่ Parameter ต่างกันใน
   Scope เดียวกัน ทั้งสองเป็นคนละแนวคิดกันโดยสิ้นเชิง

6. **คิดว่า Virtual Function ทำให้ Compile ช้าลงมากจนต้องหลีกเลี่ยงเสมอ** — ในทางปฏิบัติ
   Overhead ของ Virtual Function (การเดินทางผ่าน vptr/vtable) เล็กน้อยมากในโปรแกรมส่วนใหญ่
   ควรออกแบบโค้ดให้ถูกต้องและอ่านง่ายก่อน แล้วค่อย Optimize เฉพาะจุดที่พิสูจน์แล้วว่าเป็น
   คอขวดจริงๆ ด้วยการ Profile (เรียนใน Part 86) ไม่ใช่เดาเอาเองตั้งแต่ต้น

7. **ลืมว่า Constructor เรียก Virtual Function แบบ Dynamic Dispatch ไม่ได้** — ถ้าเรียก
   Virtual Function จากภายใน Constructor หรือ Destructor ของ Base Class มันจะเรียก Version
   ของ Class ปัจจุบันเสมอ (ไม่ใช่ Version ของ Derived Class) เพราะตอนที่ Constructor ของ
   Base ทำงาน ส่วนของ Derived ยังไม่ถูกสร้างขึ้นเลย (และตอน Destructor ของ Base ทำงาน ส่วน
   ของ Derived ก็ถูกทำลายไปแล้ว) นี่เป็นกฎที่ละเอียดอ่อนแต่สำคัญมากเมื่อออกแบบ Class Hierarchy
   ที่ซับซ้อน

---

## แบบฝึกหัดท้ายบท

1. เขียน Class `Employee` ที่มี `virtual double totalSalary() const` (คืนค่า `baseSalary`
   เฉยๆ) และ Class `Manager : public Employee` ที่ Override `totalSalary()` ให้บวก `bonus`
   เข้าไปด้วย จากนั้นเขียนฟังก์ชัน `printAnySlip(const Employee&)` ที่เรียก `totalSalary()`
   แล้วทดสอบว่าเรียกถูกต้องทั้งกับ `Employee` และ `Manager`

2. จากโค้ดในหัวข้อ 49.1 (ตัวอย่างที่ลืมใส่ `virtual`) ให้แก้ไขให้ถูกต้องโดยเพิ่ม `virtual`
   และ `override` ให้ครบถ้วน แล้วทดสอบว่าผลลัพธ์ถูกต้องตามที่คาดหวัง

3. อธิบายด้วยคำพูดของตัวเอง (เขียนเป็น comment) ว่าทำไมการเรียก Virtual Function จาก
   Constructor ของ Base Class จึงไม่ทำงานแบบ Dynamic Dispatch ตามที่คาดหวัง (ดู Pitfall ข้อ 7)
   ลองเขียนโค้ดทดสอบเพื่อพิสูจน์พฤติกรรมนี้ด้วยตัวเอง

4. เขียน Class `Resource` ที่มี `virtual ~Resource()` และ Class `FileResource : public
   Resource` ที่จองและคืน Memory ด้วยตัวเอง (`new`/`delete`) แล้วทดสอบว่าการลบผ่าน
   `Resource*` ที่ชี้ไปยัง `FileResource` เรียก Destructor ครบทั้งสองชั้นถูกต้อง

5. เขียน Class Hierarchy `Shape` → `Circle`, `Triangle` (อย่างน้อย 2 ชนิด) แล้วเก็บ Object
   ทั้งหมดไว้ใน `std::vector<std::unique_ptr<Shape>>` จากนั้นเขียน Loop ที่คำนวณผลรวมพื้นที่
   ของทุกรูปทรงในเวลาเดียวกัน โดยไม่ต้องรู้ล่วงหน้าว่า `vector` มีรูปทรงชนิดใดบ้าง

6. ทดลองเขียนโค้ดที่ส่ง Object ของ Derived Class เป็น Parameter แบบ By-Value ให้ฟังก์ชันที่รับ
   Base Class (สาธิต Object Slicing ด้วยตัวเอง) แล้วแก้ไขให้ถูกต้องโดยเปลี่ยนเป็น Reference

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <string>

class Employee {
public:
    Employee(std::string name, double baseSalary)
        : name_(std::move(name)), baseSalary_(baseSalary) {}
    virtual ~Employee() = default;

    virtual double totalSalary() const { return baseSalary_; }

    void printSlip() const {
        std::cout << name_ << ": เงินเดือนรวม = " << totalSalary() << " บาท\n";
    }

protected:
    std::string name_;
    double baseSalary_;
};

class Manager : public Employee {
public:
    Manager(std::string name, double baseSalary, double bonus)
        : Employee(std::move(name), baseSalary), bonus_(bonus) {}

    double totalSalary() const override { return baseSalary_ + bonus_; }

private:
    double bonus_;
};

void printAnySlip(const Employee& e) {   // ทำงานถูกต้องกับ Employee หรือ Manager ก็ได้
    e.printSlip();
}

int main() {
    Employee e("Somchai", 30000.0);
    Manager m("Somsri", 50000.0, 15000.0);

    printAnySlip(e);   // เรียกผ่าน Employee& -> ได้ totalSalary ของ Employee
    printAnySlip(m);   // เรียกผ่าน Employee& แต่ object จริงเป็น Manager -> ได้ totalSalary ของ Manager

    return 0;
}
```

```
Somchai: เงินเดือนรวม = 30000 บาท
Somsri: เงินเดือนรวม = 65000 บาท
```

สังเกตว่าตอนนี้ต่างจากแบบฝึกหัดข้อ 4 ของ Part 48 อย่างสิ้นเชิง: เพราะ `totalSalary()` เป็น
`virtual` แล้ว การเรียกผ่าน `const Employee&` ใน `printAnySlip()` จึงเลือก Version ที่ถูกต้อง
ตามชนิดจริงของ Object โดยอัตโนมัติเสมอ ไม่ว่า Object นั้นจะเป็น `Employee` หรือ `Manager` ก็ตาม

### แนวทางเฉลยข้อ 4

```cpp
#include <iostream>

class Resource {
public:
    Resource() { std::cout << "Resource acquired\n"; }
    virtual ~Resource() { std::cout << "Resource released\n"; }
};

class FileResource : public Resource {
public:
    FileResource() {
        buffer_ = new char[256];
        std::cout << "FileResource: จอง buffer 256 bytes\n";
    }
    ~FileResource() override {
        delete[] buffer_;
        std::cout << "FileResource: คืน buffer แล้ว\n";
    }

private:
    char* buffer_;
};

int main() {
    Resource* r = new FileResource();
    delete r;   // เพราะ ~Resource() เป็น virtual -> เรียก ~FileResource() ก่อนถูกต้อง ไม่มี leak

    return 0;
}
```

```
Resource acquired
FileResource: จอง buffer 256 bytes
FileResource: คืน buffer แล้ว
Resource released
```

สังเกตลำดับ: `~FileResource()` ถูกเรียกก่อน `~Resource()` เสมอ (Derived ก่อน Base) และเพราะ
`~Resource()` เป็น `virtual` การเรียก `delete r;` ผ่าน `Resource*` จึงหา Destructor ที่ถูกต้อง
ของ `FileResource` เจอโดยอัตโนมัติ ไม่มี Memory Leak เกิดขึ้นเลย

(สำหรับข้อ 2, 3, 5, 6: ให้ผู้เรียนลองทำเองตามแนวทางในหัวข้อ 49.1–49.2, Pitfall ข้อ 7, 49.8
และ 49.5 ของ Part นี้ตามลำดับ โดยยึดหลัก "Class ที่จะใช้แบบ Polymorphism ต้องมี `virtual`
Destructor และ Override ทุกจุดต้องมี `override`" เป็นแนวทางหลักในการตรวจสอบความถูกต้องของ
โค้ดที่เขียน)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจความแตกต่างระหว่าง Static Binding (Compile Time) กับ Dynamic Binding (Runtime)
  และเห็นบั๊กจริงที่เกิดขึ้นถ้าลืมใส่ `virtual`
- เข้าใจกลไกเบื้องหลังของ Virtual Function ผ่าน vtable และ vptr อย่างละเอียด
- ใช้ `override` (C++11) เพื่อให้ Compiler ช่วยจับบั๊กจาก Signature ไม่ตรงกันตั้งแต่ Compile
  Time
- เข้าใจปัญหา Object Slicing และรู้วิธีป้องกันด้วยการใช้ Reference/Pointer เสมอ
- เข้าใจอย่างลึกซึ้งว่าทำไม Virtual Destructor ถึงสำคัญมาก และเห็น Memory Leak จริงที่เกิดขึ้น
  เมื่อลืมใส่
- ใช้ `final` เพื่อล็อกไม่ให้ Method หรือ Class ถูก Override/Inherit ต่อ
- เห็นภาพรวมทั้งหมดผ่านตัวอย่าง Shape Hierarchy ที่เก็บ Object หลายชนิดผ่าน Base Pointer
  ใน Container เดียวกัน

Polymorphism คือเสาหลักต้นที่สามของ OOP ที่ทำให้โค้ดของเรา **ขยายได้โดยไม่ต้องแก้ไขโค้ดเดิม**
(Open-Closed Principle) แต่สิ่งที่เรายังไม่ได้พูดถึงคือ: ถ้า `Shape::area()` ในตัวอย่างที่ผ่านมา
ไม่ควรมี "ค่าเริ่มต้น" เลย (เพราะ `Shape` ที่เป็นนามธรรมไม่ควรถูกสร้างเป็น Object ได้โดยตรง)
เราจะบังคับให้ Class ลูกทุกตัว **ต้อง** Override ได้อย่างไร? นี่คือคำถามที่ **Part 50 —
Abstract Class และ Interface** จะตอบ ผ่านแนวคิดของ **Pure Virtual Function** และการออกแบบ
Interface ที่ดีสำหรับ C++

**ต่อไป:** [Part 50 — Abstract Class และ Interface](./part-050-abstract-classes.md)
