# Part 45: เริ่มต้น Class และ Object (Step 353–360)

> Module D — เริ่มต้น C++ และ OOP | Part 45 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 353–360
> Part ก่อนหน้า: [Part 44 — ฟังก์ชันใน C++ (Overload/Default/inline)](./part-044-functions-cpp.md) | Part ถัดไป: [Part 46 — Constructor และ Destructor](./part-046-constructors-destructors.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายแนวคิดพื้นฐานของ **Object-Oriented Programming (OOP)** โดยเฉพาะหลักการ
   **Encapsulation** (การห่อหุ้มข้อมูลและพฤติกรรมไว้ด้วยกัน) และบอกได้ว่าทำไม C ถึงทำสิ่งนี้
   ได้ไม่สมบูรณ์เท่า C++
2. แยกความแตกต่างระหว่าง **`class`** กับ **`struct`** ใน C++ และอธิบายได้ว่าความต่างเดียว
   ที่แท้จริงคือ **default access specifier**
3. ประกาศ **member variable** และ **member function** ภายใน class ได้อย่างถูกต้อง
4. สร้าง **object** จาก class ได้หลายรูปแบบ และเข้าใจว่าแต่ละ object มีสำเนาของ member
   variable เป็นของตัวเอง
5. อธิบายและใช้งาน **`this` pointer** ได้อย่างถูกต้อง โดยเฉพาะกรณีพารามิเตอร์ชื่อชนกับ
   member variable
6. แยก **class declaration** (`.h`) ออกจาก **implementation** (`.cpp`) ได้ตาม convention
   มาตรฐานที่ใช้ในโปรเจกต์ C++ จริง
7. ออกแบบและเขียน class ที่ใช้งานได้จริงอย่าง `Rectangle` และ `BankAccount` ตั้งแต่ต้นจนจบ

---

## 45.1 แนวคิด OOP เบื้องต้น: Encapsulation (Step 353)

ตลอด Module A ถึง C เราเขียนโปรแกรมด้วยแนวคิด **Procedural Programming**: แยก **ข้อมูล**
(struct, array, ตัวแปร) ออกจาก **พฤติกรรม/ฟังก์ชันที่กระทำต่อข้อมูลนั้น** อย่างชัดเจน เช่นใน C
ถ้าเรามี `struct Rectangle { double width; double height; };` ฟังก์ชันที่คำนวณพื้นที่ก็ต้อง
แยกเขียนต่างหากเป็น `double rectangle_area(struct Rectangle r);` — ไม่มีอะไรผูก struct กับ
ฟังก์ชันเข้าด้วยกันอย่างเป็นทางการเลย ผู้เขียนโค้ดต้องจำเอาเองว่า "ฟังก์ชันไหนใช้กับ struct ไหน"

**Object-Oriented Programming (OOP)** เสนอแนวคิดใหม่ที่เรียกว่า **Encapsulation**
(การห่อหุ้ม): แทนที่จะแยกข้อมูลกับฟังก์ชันออกจากกัน ให้ **รวมข้อมูล (state) และพฤติกรรม
(behavior) ที่เกี่ยวข้องกันไว้เป็นหน่วยเดียวกัน** เรียกหน่วยนี้ว่า **class** — object ที่สร้าง
จาก class จะ "รู้ตัวเองว่าทำอะไรได้บ้าง" โดยไม่ต้องพึ่งฟังก์ชันข้างนอกที่กระจัดกระจาย

```
แนวคิดแบบ Procedural (C):          แนวคิดแบบ OOP (C++):
┌─────────────────┐                ┌──────────────────────────┐
│ struct Rectangle │                │ class Rectangle           │
│  - width         │                │  - width  (ข้อมูล)        │
│  - height        │                │  - height (ข้อมูล)        │
└─────────────────┘                │  - area()      (พฤติกรรม) │
        ▲                          │  - perimeter() (พฤติกรรม) │
        │ ใช้ร่วมกับ                │  - scale()     (พฤติกรรม) │
┌─────────────────┐                └──────────────────────────┘
│ rectangle_area() │                     ข้อมูล + พฤติกรรม
│ rectangle_perim()│                     อยู่ในกล่องเดียวกัน
└─────────────────┘
```

ประโยชน์หลักของ Encapsulation มีอย่างน้อย 3 ข้อ:

1. **จัดระเบียบโค้ด**: เปิดไฟล์ class เดียว เห็นครบทั้งข้อมูลและพฤติกรรมที่เกี่ยวข้อง ไม่ต้อง
   ไล่หาฟังก์ชันที่กระจัดกระจายอยู่หลายไฟล์
2. **ควบคุมการเข้าถึงข้อมูล (Access Control)**: สามารถ "ซ่อน" รายละเอียดภายในไม่ให้ผู้ใช้
   class เข้าถึงโดยตรง (ผ่าน `private`/`public` ที่จะเจาะลึกใน Part 47) บังคับให้ต้องแก้ไข
   ข้อมูลผ่าน method ที่ตรวจสอบความถูกต้องได้ (เช่น `deposit()` ที่เช็คว่าจำนวนเงินต้องเป็นบวก)
3. **นำไปต่อยอดได้**: เป็นรากฐานของแนวคิด OOP ที่เหลือทั้งหมด (Inheritance ใน Part 48,
   Polymorphism ใน Part 49) ซึ่งจะเรียนต่อเนื่องกันไปตลอด Module D

> **หมายเหตุ**: C++ ไม่ได้ "แทนที่" Procedural Programming เสียทีเดียว — โค้ด C++ จริงมักผสม
> ทั้งสองแนวคิดเข้าด้วยกัน ฟังก์ชัน `main()` เองก็ยังเป็นแนวคิด procedural แต่ตัว logic ภายใน
> มักถูกจัดกลุ่มเป็น class ตามความเหมาะสม

---

## 45.2 class vs struct ใน C++: ความต่างเดียวคือ Default Access (Step 354)

ใน C, `struct` ใช้ได้แค่เก็บกลุ่มข้อมูลเท่านั้น (ไม่มี `class` ให้ใช้เลย) แต่ **ใน C++, `struct`
ถูกยกระดับให้มีความสามารถเกือบทั้งหมดเหมือน `class`** — มี member function ได้, มี
constructor ได้ (จะเรียนใน Part 46), มี access specifier ได้ครบ (`public`, `private`,
`protected` — จะเรียนใน Part 47)

คำถามที่ตามมาคือ: ถ้า `struct` กับ `class` ทำอะไรได้เหมือนกันหมดแล้ว ต่างกันตรงไหน?

**คำตอบคือมีความต่างเดียวเท่านั้น: default access specifier**

| | `struct` | `class` |
|---|---|---|
| Default access ของ member | `public` (เข้าถึงจากภายนอกได้ทันที) | `private` (เข้าถึงจากภายนอกไม่ได้ ต้องผ่าน public method) |
| Default access ของการสืบทอด (inheritance, Part 48) | `public` | `private` |
| ความสามารถอื่นๆ ทั้งหมด | เหมือนกันทุกประการ | เหมือนกันทุกประการ |

ลองดูตัวอย่างเปรียบเทียบตรงๆ:

```cpp
// class_vs_struct.cpp
#include <iostream>

struct PointStruct {
    double x;
    double y;
};

class PointClass {
    double x;   // ไม่ระบุ access specifier แปลว่า private โดย default
    double y;
public:
    PointClass(double x_val, double y_val) : x(x_val), y(y_val) {}
    double getX() const { return x; }
    double getY() const { return y; }
};

int main() {
    PointStruct ps{1.0, 2.0};   // struct: member เป็น public โดย default เข้าถึงตรงๆ ได้
    std::cout << "struct: (" << ps.x << ", " << ps.y << ")\n";

    PointClass pc(3.0, 4.0);    // class: member เป็น private โดย default ต้องผ่าน public method
    std::cout << "class: (" << pc.getX() << ", " << pc.getY() << ")\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 class_vs_struct.cpp -o class_vs_struct
./class_vs_struct
```

```
struct: (1, 2)
class: (3, 4)
```

ลองพิสูจน์ว่า member ของ `class` เป็น `private` จริงๆ ด้วยการพยายามเข้าถึงโดยตรงจากภายนอก:

```cpp
// private_error.cpp — ตัวอย่างนี้ "ผิดโดยตั้งใจ" เพื่อสาธิต error ของการเข้าถึง private member
#include <iostream>

class PointClass {
    double x;
    double y;
public:
    PointClass(double x_val, double y_val) : x(x_val), y(y_val) {}
};

int main() {
    PointClass pc(3.0, 4.0);
    std::cout << pc.x << '\n'; // ERROR: x เป็น private เข้าถึงจากนอก class ไม่ได้
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 private_error.cpp -o private_error
```

```
error: 'double PointClass::x' is private within this context
note: declared private here
```

compiler ปฏิเสธการเข้าถึง `pc.x` ตรงๆ ทันที เพราะ `x` เป็น `private` (ค่า default ของ
`class`) — ถ้าเปลี่ยน `class PointClass` เป็น `struct PointClass` โดยไม่แก้อะไรอย่างอื่นเลย
โค้ดชุดนี้จะ **compile ผ่านทันที** เพราะ default เปลี่ยนเป็น `public`

> **ธรรมเนียมการเลือกใช้ (Convention) ในโค้ดจริง**: แม้ทางเทคนิคจะสลับกันใช้ได้ แต่วงการ C++
> มีธรรมเนียมไม่เป็นทางการที่ยึดถือกันแทบทุกที่:
> - ใช้ **`struct`** สำหรับกลุ่มข้อมูลล้วนๆ ที่ไม่มี invariant ต้องรักษา (Plain Old Data /
>   POD) เช่น `struct Point3D { double x, y, z; };` — เน้นให้เข้าถึง member ได้ตรงๆ
> - ใช้ **`class`** เมื่อต้องการห่อหุ้มข้อมูล ควบคุมการเข้าถึง หรือมี invariant ที่ต้องรักษา
>   (เช่น ยอดเงินในบัญชีต้องไม่ติดลบ) — เน้นซ่อนรายละเอียดภายในไว้เป็น private
>
> Part นี้และ Part ถัดๆ ไปในหลักสูตรจะใช้ `class` เป็นหลักเมื่อออกแบบ object ที่มีพฤติกรรม
> ซับซ้อน และจะเจาะลึกเรื่อง access control อย่างเต็มรูปแบบใน **Part 47**

---

## 45.3 การประกาศ Member Variable และ Member Function (Step 355)

**Member Variable** (บางตำราเรียก **Data Member** หรือ **Field**) คือตัวแปรที่ประกาศอยู่
ภายใน class — เก็บ "สถานะ (state)" ของ object แต่ละตัว

**Member Function** (บางตำราเรียก **Method**) คือฟังก์ชันที่ประกาศอยู่ภายใน class — กำหนด
"พฤติกรรม (behavior)" ที่ object ทำได้ Member function มีสิทธิ์เข้าถึง member variable ของ
object เดียวกันได้โดยตรง โดยไม่ต้องส่งเป็นพารามิเตอร์เหมือนฟังก์ชันธรรมดาใน C

```cpp
// dog.cpp
#include <iostream>
#include <string>

class Dog {
public:
    std::string name;   // member variable
    int age;             // member variable

    void bark() const {  // member function
        std::cout << name << " เห่า: โฮ่ง โฮ่ง!\n";
    }

    void printInfo() const {  // member function
        std::cout << "ชื่อ: " << name << ", อายุ: " << age << " ปี\n";
    }
};

int main() {
    Dog dog1;
    dog1.name = "โบ๊ะ";
    dog1.age = 3;

    Dog dog2;
    dog2.name = "มะลิ";
    dog2.age = 5;

    dog1.printInfo();
    dog1.bark();

    dog2.printInfo();
    dog2.bark();

    // แต่ละ object มี member variable เป็นของตัวเอง แยกจากกันโดยสิ้นเชิง
    std::cout << "dog1.name = " << dog1.name << ", dog2.name = " << dog2.name << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 dog.cpp -o dog
./dog
```

```
ชื่อ: โบ๊ะ, อายุ: 3 ปี
โบ๊ะ เห่า: โฮ่ง โฮ่ง!
ชื่อ: มะลิ, อายุ: 5 ปี
มะลิ เห่า: โฮ่ง โฮ่ง!
dog1.name = โบ๊ะ, dog2.name = มะลิ
```

สังเกตประเด็นสำคัญ 2 ข้อจากตัวอย่างนี้:

1. **`bark()` และ `printInfo()` เข้าถึง `name` กับ `age` ได้โดยตรง** โดยไม่ต้องรับเป็น
   พารามิเตอร์เลย ต่างจากฟังก์ชันแบบ C ที่ต้องเขียน `void bark(struct Dog d)` แล้วเข้าถึงผ่าน
   `d.name` เสมอ — เพราะ member function "รู้อยู่แล้ว" ว่ากำลังทำงานกับ object ตัวไหน (ผ่าน
   `this` pointer ที่จะอธิบายในหัวข้อ 45.5)
2. **`dog1` และ `dog2` มี `name`/`age` แยกจากกันคนละชุด** แม้จะสร้างจาก class เดียวกัน
   การเปลี่ยนค่าใน `dog1` ไม่มีผลต่อ `dog2` เลย — นี่คือธรรมชาติพื้นฐานของ object: class คือ
   "พิมพ์เขียว (blueprint)" ส่วน object คือ "ของจริงที่สร้างจากพิมพ์เขียวนั้น" แต่ละ object
   มีหน่วยความจำสำหรับ member variable เป็นของตัวเอง (ในขณะที่ member function ใช้โค้ด
   ชุดเดียวกันร่วมกันทุก object ไม่ได้ถูกคัดลอกซ้ำ)

### `const` ท้าย member function คืออะไร

สังเกตว่า `bark()` และ `printInfo()` มี `const` ต่อท้าย signature — นี่คือการบอก compiler ว่า
"member function นี้จะไม่แก้ไข member variable ใดๆ ของ object เลย" ถ้าเผลอเขียนโค้ดที่แก้ไข
member variable ภายในฟังก์ชันที่ประกาศ `const` ไว้ compiler จะ error ทันที นี่คือการนำแนวคิด
**const correctness** จาก Part 43 มาประยุกต์ใช้กับ member function โดยตรง และจะกลายเป็น
ธรรมเนียมสำคัญที่ใช้ตลอดหลักสูตรที่เหลือ: **member function ที่ไม่แก้ไข state ของ object
ควรประกาศ `const` เสมอ**

---

## 45.4 การสร้าง Object จาก Class (Step 356)

**Object** คือค่าที่เกิดจากการสร้างตัวแปรตาม "พิมพ์เขียว" ที่ class กำหนดไว้ กระบวนการนี้
เรียกว่า **Instantiation** และ object แต่ละตัวเรียกว่า **Instance**

จากตัวอย่าง `Dog` ในหัวข้อก่อนหน้า การเขียน:

```cpp
Dog dog1;
```

คือการสร้าง object ชื่อ `dog1` จาก class `Dog` บน **stack** (เหมือนตัวแปรธรรมดาที่เรียนมา
ตั้งแต่ Part 2) — เมื่อออกจาก scope object จะถูกทำลายอัตโนมัติ (จะอธิบายกลไกนี้ผ่าน
**destructor** อย่างละเอียดใน Part 46)

การสร้างหลาย object จาก class เดียวกันทำได้อย่างอิสระ:

```cpp
Dog dog1;
Dog dog2;
Dog dog3;
```

แต่ละตัวจะมีหน่วยความจำสำหรับ `name` และ `age` แยกจากกันโดยสิ้นเชิง เปรียบเทียบได้กับการมี
"พิมพ์เขียวบ้าน" หนึ่งแบบ (class) แต่สร้างบ้านจริงได้หลายหลัง (object) — แต่ละหลังมีเจ้าของ
มีเฟอร์นิเจอร์เป็นของตัวเอง แม้จะสร้างจากพิมพ์เขียวเดียวกันก็ตาม

> **หมายเหตุสำหรับอนาคต**: Object ยังสามารถสร้างบน **heap** ได้ด้วย `new` (เช่น
> `Dog* dog4 = new Dog();`) เหมือนที่เรียน dynamic memory ใน Part 11 แต่หลักสูตรนี้จะเลื่อน
> การใช้ raw `new`/`delete` กับ object ไปพูดอย่างละเอียดพร้อมกับเรื่อง RAII และ destructor
> ใน **Part 46** และจะแนะนำ smart pointer ที่ปลอดภัยกว่าใน **Part 67** ต่อไป

---

## 45.5 `this` Pointer (Step 357)

ทุกครั้งที่เรียก member function ผ่าน object (เช่น `dog1.bark()`) compiler จะแอบส่ง
**pointer ที่ชี้ไปยัง object นั้น** เข้าไปในฟังก์ชันโดยอัตโนมัติ pointer ตัวนี้มีชื่อพิเศษว่า
**`this`** และสามารถใช้งานได้จากภายใน member function ทุกตัว (ยกเว้น `static` member
function ที่จะเรียนใน Part 53 เพราะไม่ได้ผูกกับ object ตัวใดตัวหนึ่ง)

```cpp
// this_ptr.cpp
#include <iostream>

class Counter {
public:
    int value;

    void showAddress() const {
        // this คือ pointer ที่ชี้ไปยัง object ที่กำลังเรียก member function นี้อยู่
        std::cout << "this ชี้ไปที่ address: " << this << ", value = " << this->value << '\n';
    }

    Counter& increment() {
        this->value += 1;
        return *this; // คืนค่า object ปัจจุบัน (dereference this) เพื่อให้ chain เรียกต่อได้
    }
};

int main() {
    Counter c1;
    c1.value = 0;
    std::cout << "address ของ c1 จริงๆ: " << &c1 << '\n';
    c1.showAddress(); // this ข้างในควรเป็น address เดียวกับ &c1

    c1.increment().increment().increment(); // เรียกต่อกันได้เพราะ increment() คืน Counter&
    std::cout << "หลังเรียก increment() 3 ครั้ง: value = " << c1.value << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 this_ptr.cpp -o this_ptr
./this_ptr
```

ผลลัพธ์ (ค่า address จะต่างกันไปในแต่ละครั้งที่รัน):

```
address ของ c1 จริงๆ: 0x7ffe64eb7194
this ชี้ไปที่ address: 0x7ffe64eb7194, value = 0
หลังเรียก increment() 3 ครั้ง: value = 3
```

จะเห็นว่า address ที่ `&c1` ให้มา กับ address ที่ `this` ชี้ไปข้างในฟังก์ชัน `showAddress()`
**เป็น address เดียวกันเป๊ะ** — พิสูจน์ว่า `this` คือ pointer ที่ชี้กลับไปยัง object เดิมที่
เรียกฟังก์ชันนั้นจริงๆ

`increment()` แสดงเทคนิคที่เรียกว่า **Method Chaining**: การ `return *this;` (dereference
`this` เพื่อคืนค่าเป็น reference ของ object ปัจจุบัน) ทำให้เรียก method ต่อกันเป็นสายได้ เช่น
`c1.increment().increment().increment();` — เทคนิคนี้จะพบบ่อยมากในไลบรารีสมัยใหม่ (เช่น
`std::cout << a << b << c;` ก็ใช้หลักการเดียวกัน)

### เหตุผลสำคัญที่สุดที่ต้องใช้ `this`: แก้ปัญหาชื่อพารามิเตอร์ชนกับ member variable

สถานการณ์ที่พบบ่อยที่สุดที่จำเป็นต้องใช้ `this` อย่างชัดเจนคือเมื่อ **ชื่อพารามิเตอร์ตรงกับ
ชื่อ member variable พอดี** (ซึ่งเป็นแบบแผนที่นิยมมาก เพราะทำให้โค้ดอ่านง่าย ไม่ต้องคิดชื่อ
แยกให้ยุ่งยาก เช่น `setWidth(double width)` แทนที่จะตั้งชื่อพารามิเตอร์แปลกๆ อย่าง `newW`)

```cpp
// shadow_bug.cpp
#include <iostream>

class Point {
public:
    double x;
    double y;

    // ตัวอย่างนี้ "จงใจ" เขียนบั๊ก: ลืมใช้ this-> ทำให้ x = x เป็นการกำหนดค่าพารามิเตอร์ให้ตัวเอง
    // ไม่ได้กำหนดให้ member x เลย เพราะพารามิเตอร์ x บัง (shadow) member x ไว้ในขอบเขตนี้
    void setX_buggy(double x) {
        x = x; // ไม่มีผลใดๆ ต่อ member x เลย!
    }

    void setX_correct(double x) {
        this->x = x; // this->x คือ member ชัดเจน แก้ปัญหา shadowing ได้ทันที
    }
};

int main() {
    Point p;
    p.x = 100.0;

    p.setX_buggy(999.0);
    std::cout << "หลัง setX_buggy(999.0): p.x = " << p.x << " (ยังเป็น 100 เพราะบั๊ก)\n";

    p.setX_correct(999.0);
    std::cout << "หลัง setX_correct(999.0): p.x = " << p.x << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 shadow_bug.cpp -o shadow_bug
./shadow_bug
```

```
หลัง setX_buggy(999.0): p.x = 100 (ยังเป็น 100 เพราะบั๊ก)
หลัง setX_correct(999.0): p.x = 999
```

เหตุผลของบั๊กนี้เกี่ยวข้องกับ **Scope** (ที่เรียนมาตั้งแต่ Part 6): ภายใน `setX_buggy`,
ชื่อ `x` ที่ใกล้ที่สุด (พารามิเตอร์) จะ **บัง (shadow)** ชื่อ `x` ที่เป็น member variable ไว้
ทำให้ `x = x;` กลายเป็นการกำหนดค่าพารามิเตอร์ให้ตัวมันเอง ไม่มีผลใดๆ ต่อ member ของ object
เลย การเขียน `this->x = x;` แก้ปัญหานี้ได้เด็ดขาด เพราะ `this->x` ระบุชัดเจนว่าหมายถึง
member variable ของ object เท่านั้น ไม่ปนกับพารามิเตอร์

---

## 45.6 แยก Class Declaration (.h) ออกจาก Implementation (.cpp) (Step 358)

เช่นเดียวกับฟังก์ชันธรรมดาที่เรียนใน Part 17 (Modular Programming) การเขียน class ในโปรเจกต์
จริงก็แยก 2 ส่วนออกจากกัน:

- **Header (`.h`)**: ประกาศ **โครงสร้าง (interface)** ของ class — มี member variable และ
  **prototype** ของ member function เท่านั้น (ไม่มี implementation จริง)
- **Implementation (`.cpp`)**: เขียน **โค้ดจริง** ของ member function แต่ละตัว โดยใช้ scope
  resolution operator `::` เพื่อบอกว่าฟังก์ชันนี้เป็นของ class ไหน

```cpp
// Rectangle.h
#ifndef RECTANGLE_H
#define RECTANGLE_H

class Rectangle {
private:
    double width;
    double height;

public:
    Rectangle(double width, double height);

    double area() const;
    double perimeter() const;
    void setWidth(double newWidth);
    void setHeight(double newHeight);
    double getWidth() const;
    double getHeight() const;
};

#endif
```

```cpp
// Rectangle.cpp
#include "Rectangle.h"

Rectangle::Rectangle(double width, double height) {
    this->width = width;
    this->height = height;
}

double Rectangle::area() const {
    return width * height;
}

double Rectangle::perimeter() const {
    return 2 * (width + height);
}

void Rectangle::setWidth(double newWidth) {
    width = newWidth;
}

void Rectangle::setHeight(double newHeight) {
    height = newHeight;
}

double Rectangle::getWidth() const {
    return width;
}

double Rectangle::getHeight() const {
    return height;
}
```

```cpp
// rectangle_main.cpp
#include <iostream>
#include "Rectangle.h"

int main() {
    Rectangle r(4.0, 5.0);
    std::cout << "area = " << r.area() << ", perimeter = " << r.perimeter() << '\n';

    r.setWidth(10.0);
    std::cout << "หลัง setWidth(10.0): width = " << r.getWidth()
              << ", area = " << r.area() << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 rectangle_main.cpp Rectangle.cpp -o rectangle_main
./rectangle_main
```

```
area = 20, perimeter = 18
หลัง setWidth(10.0): width = 10, area = 50
```

สังเกต `Rectangle::area()` ใน `Rectangle.cpp` — เครื่องหมาย `::` (Scope Resolution
Operator) บอก compiler ว่า "ฟังก์ชัน `area` ตัวนี้เป็น member ของ class `Rectangle`"
ไม่ใช่ฟังก์ชันอิสระธรรมดา ภายใน implementation นี้จึงเข้าถึง `width`/`height` (ซึ่งเป็น
`private` member) ได้โดยตรง ทั้งที่ประกาศไว้คนละไฟล์กับ header

### ทำไมต้องแยกไฟล์แบบนี้

เหตุผลเดียวกับที่เรียนใน Part 17: **ผู้ใช้ class ไม่จำเป็นต้องเห็นรายละเอียดการ implement**
เขาแค่ `#include "Rectangle.h"` เพื่อรู้ว่า class นี้ "ทำอะไรได้บ้าง" (interface) โดยไม่ต้อง
สนใจว่าข้างในคำนวณอย่างไร — เมื่อแก้ไข logic ภายใน `Rectangle.cpp` ผู้ใช้ class ก็ไม่ต้อง
compile ไฟล์ที่ `#include` header นี้ใหม่ทั้งหมด (แค่ compile `Rectangle.cpp` ใหม่ แล้ว link
เข้าด้วยกัน) ช่วยลดเวลา compile ของโปรเจกต์ขนาดใหญ่ได้มาก และยังเป็นการ **ซ่อนรายละเอียด
การ implement** (Information Hiding) ซึ่งเป็นหัวใจของ Encapsulation ที่เรียนในหัวข้อ 45.1

> ตลอด Module D เป็นต้นไป หลักสูตรนี้จะยึดรูปแบบการแยกไฟล์นี้เป็นมาตรฐานสำหรับ class ที่มี
> ขนาดใหญ่พอสมควร ส่วน class เล็กๆ ที่ใช้สาธิตแนวคิดในบทเรียน อาจยังคงเขียน implementation
> ไว้ใน class โดยตรง (เรียกว่า **inline definition ภายใน class**) เพื่อความกระชับของตัวอย่าง

---

## 45.7 ตัวอย่างจริง: class Rectangle แบบสมบูรณ์ (Step 359)

หัวข้อ 45.6 ได้แนะนำ `Rectangle` ไปแล้วในรูปแบบแยกไฟล์ ในหัวข้อนี้เราจะดูอีกเวอร์ชันหนึ่งที่
เขียน implementation ไว้ในตัว class โดยตรง (inline) เพื่อเห็นภาพรวมทุก concept ที่เรียนมา
ใน Part นี้ — member variable, member function, `this`, และ method ที่รับ object อื่นเป็น
พารามิเตอร์ (ทบทวนเรื่อง const reference จาก Part 44) — ในไฟล์เดียว:

```cpp
// rectangle_inline.cpp
#include <iostream>

class Rectangle {
public:
    double width;
    double height;

    double area() const {
        return width * height;
    }

    double perimeter() const {
        return 2 * (width + height);
    }

    void setWidth(double width) {
        this->width = width; // this->width คือ member, width (ไม่มี this->) คือพารามิเตอร์
    }

    void setHeight(double height) {
        this->height = height;
    }

    void scale(double factor) {
        this->width *= factor;
        this->height *= factor;
    }

    bool isLargerThan(const Rectangle& other) const {
        return this->area() > other.area(); // this ใช้เปรียบเทียบ object ปัจจุบันกับตัวอื่น
    }
};

int main() {
    Rectangle r1;
    r1.width = 4.0;
    r1.height = 5.0;

    std::cout << "r1 area = " << r1.area() << ", perimeter = " << r1.perimeter() << '\n';

    r1.setWidth(10.0);
    std::cout << "หลัง setWidth(10.0): area = " << r1.area() << '\n';

    r1.scale(2.0);
    std::cout << "หลัง scale(2.0): width=" << r1.width << " height=" << r1.height << '\n';

    Rectangle r2;
    r2.width = 2.0;
    r2.height = 2.0;

    std::cout << "r1 ใหญ่กว่า r2 หรือไม่: " << (r1.isLargerThan(r2) ? "ใช่" : "ไม่ใช่") << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 rectangle_inline.cpp -o rectangle_inline
./rectangle_inline
```

```
r1 area = 20, perimeter = 18
หลัง setWidth(10.0): area = 50
หลัง scale(2.0): width=20 height=10
r1 ใหญ่กว่า r2 หรือไม่: ใช่
```

`isLargerThan(const Rectangle& other)` เป็นตัวอย่างที่ดีของการผสมผสานความรู้จากหลาย Part:
รับพารามิเตอร์เป็น `const Rectangle&` (Part 44: type ใหญ่ที่แค่ต้องการอ่าน ใช้ const
reference) แล้วเปรียบเทียบ `this->area()` (พื้นที่ของ object ปัจจุบัน) กับ `other.area()`
(พื้นที่ของ object ที่ส่งเข้ามา) — สังเกตว่าเรียก `this->area()` หรือเขียนแค่ `area()` เฉยๆ
ก็ได้ผลเหมือนกัน (compiler เติม `this->` ให้อัตโนมัติเมื่อเราไม่ได้ระบุ) แต่การเขียน
`this->area()` ให้ชัดเจนช่วยให้ผู้อ่านโค้ดแยกแยะระหว่าง "เรียก method ของ object ปัจจุบัน"
กับ "เรียก method ของ object อื่นที่ส่งเข้ามา" (`other.area()`) ได้ง่ายขึ้น

---

## 45.8 ตัวอย่างจริง: class BankAccount แบบสมบูรณ์ (Step 360)

ปิดท้าย Part นี้ด้วยตัวอย่างที่ใกล้เคียงการใช้งานจริงมากขึ้น: `BankAccount` ที่แสดงให้เห็น
ประโยชน์ของ Encapsulation อย่างชัดเจนที่สุด — เราไม่อยากให้ใครแก้ไข `balance` ตรงๆ ได้
(เพราะอาจตั้งเป็นค่าติดลบมั่วๆ) จึงบังคับให้แก้ไขผ่าน `deposit()`/`withdraw()` ที่มีการ
ตรวจสอบความถูกต้องเสมอ

```cpp
// BankAccount.h
#ifndef BANK_ACCOUNT_H
#define BANK_ACCOUNT_H

#include <string>

class BankAccount {
private:
    std::string ownerName;
    double balance;

public:
    BankAccount(const std::string& ownerName, double initialBalance);

    void deposit(double amount);
    bool withdraw(double amount);
    double getBalance() const;
    const std::string& getOwnerName() const;
};

#endif
```

```cpp
// BankAccount.cpp
#include "BankAccount.h"
#include <iostream>

BankAccount::BankAccount(const std::string& ownerName, double initialBalance) {
    this->ownerName = ownerName;
    this->balance = initialBalance;
}

void BankAccount::deposit(double amount) {
    if (amount <= 0) {
        std::cout << "จำนวนเงินฝากต้องมากกว่า 0\n";
        return;
    }
    balance += amount;
}

bool BankAccount::withdraw(double amount) {
    if (amount <= 0) {
        std::cout << "จำนวนเงินถอนต้องมากกว่า 0\n";
        return false;
    }
    if (amount > balance) {
        std::cout << "ยอดเงินไม่พอ (มีอยู่ " << balance << " บาท)\n";
        return false;
    }
    balance -= amount;
    return true;
}

double BankAccount::getBalance() const {
    return balance;
}

const std::string& BankAccount::getOwnerName() const {
    return ownerName;
}
```

```cpp
// bank_main.cpp
#include <iostream>
#include "BankAccount.h"

int main() {
    BankAccount acc("สมชาย ใจดี", 1000.0);

    std::cout << acc.getOwnerName() << " มีเงิน " << acc.getBalance() << " บาท\n";

    acc.deposit(500.0);
    std::cout << "หลังฝาก 500: " << acc.getBalance() << " บาท\n";

    bool ok = acc.withdraw(2000.0);
    std::cout << "ถอน 2000 สำเร็จหรือไม่: " << (ok ? "สำเร็จ" : "ไม่สำเร็จ") << '\n';

    ok = acc.withdraw(300.0);
    std::cout << "ถอน 300 สำเร็จหรือไม่: " << (ok ? "สำเร็จ" : "ไม่สำเร็จ") << '\n';
    std::cout << "ยอดเงินคงเหลือ: " << acc.getBalance() << " บาท\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 bank_main.cpp BankAccount.cpp -o bank_main
./bank_main
```

```
สมชาย ใจดี มีเงิน 1000 บาท
หลังฝาก 500: 1500 บาท
ยอดเงินไม่พอ (มีอยู่ 1500 บาท)
ถอน 2000 สำเร็จหรือไม่: ไม่สำเร็จ
ถอน 300 สำเร็จหรือไม่: สำเร็จ
ยอดเงินคงเหลือ: 1200 บาท
```

สังเกตว่า **ไม่มีทางเข้าถึง `acc.balance` โดยตรงจากภายนอกได้เลย** เพราะเป็น `private`
ผู้ใช้ `BankAccount` ถูกบังคับให้ผ่าน `deposit()`/`withdraw()` เท่านั้น ซึ่งทั้งสอง method นี้
มีการตรวจสอบเงื่อนไข (จำนวนเงินต้องเป็นบวก, ยอดถอนต้องไม่เกินยอดคงเหลือ) ก่อนแก้ไข `balance`
ทุกครั้ง — นี่คือสิ่งที่ struct ธรรมดาใน C ทำไม่ได้เลย (ใครก็แก้ `balance` ตรงๆ ให้เป็นค่า
ติดลบมั่วๆ ก็ได้ ถ้าไม่มีวินัยในการเรียกฟังก์ชันตรวจสอบก่อนเสมอ) และคือเหตุผลที่แท้จริงว่าทำไม
Encapsulation ถึงสำคัญในการออกแบบซอฟต์แวร์จริง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เข้าใจผิดว่า `struct` กับ `class` ต่างกันในเชิงความสามารถ** — ทั้งสองทำได้เหมือนกันทุก
   ประการ (member function, constructor, inheritance ฯลฯ) ต่างกันแค่ **default access
   specifier** เท่านั้น (`struct` = `public`, `class` = `private`)
2. **ลืมว่า member ของ `class` เป็น `private` โดย default** — ถ้าประกาศ member variable/
   function โดยไม่ระบุ `public:` ไว้ก่อน แล้วพยายามเข้าถึงจากภายนอก จะได้ compile error
   "is private within this context" ทันที ต้องระบุ `public:` ให้ชัดเจนถ้าต้องการให้เข้าถึง
   จากภายนอกได้
3. **ตั้งชื่อพารามิเตอร์ใน setter ตรงกับชื่อ member variable แล้วลืมใช้ `this->`** — ทำให้เกิด
   shadowing ที่การกำหนดค่าไม่มีผลใดๆ ต่อ member ของ object เลย (เช่น `x = x;` กลายเป็นกำหนด
   ค่าพารามิเตอร์ให้ตัวเอง) ต้องเขียน `this->x = x;` เพื่อระบุ member ให้ชัดเจนเสมอเมื่อชื่อชนกัน
4. **ลืมใส่ `const` ท้าย member function ที่ไม่ได้แก้ไข state** — แม้จะไม่ error ทันที แต่จะ
   สร้างปัญหาภายหลังเมื่อมีใครพยายามเรียก method นั้นผ่าน `const Rectangle&` (เช่นพารามิเตอร์
   `other` ในตัวอย่าง `isLargerThan`) เพราะ compiler จะไม่อนุญาตให้เรียก non-const method
   ผ่าน const reference/object ได้เลย
5. **สับสนระหว่าง class (พิมพ์เขียว) กับ object (ของจริงที่สร้างขึ้น)** — `class Dog { ... };`
   เป็นแค่ "นิยาม" ยังไม่ใช้หน่วยความจำสำหรับข้อมูลจริง ต้องเขียน `Dog dog1;` ก่อนถึงจะได้
   object จริงที่มี memory เป็นของตัวเอง
6. **ลืมใส่ header guard (`#ifndef`/`#define`/`#endif`) ในไฟล์ `.h` ของ class** — ถ้า header
   ถูก `#include` ซ้ำในหลายไฟล์ที่มารวมกันตอน compile เดียวกัน จะเกิด error "redefinition of
   class" ทันที (ทบทวนเรื่องนี้ได้จาก Part 14)
7. **แก้ไข implementation ใน `.cpp` แต่ประกาศ signature ใน `.h` ไม่ตรงกัน** (เช่น ลืมใส่
   `const` ท้าย method ให้ตรงกันทั้งสองไฟล์) — จะได้ linker error "undefined reference"
   เพราะ compiler มองว่าเป็นคนละฟังก์ชันกัน (สัญญาณเดียวกับที่เจอตอนเรียน function
   declaration ใน Part 6 และ 17)

---

## แบบฝึกหัดท้ายบท

1. เขียน class ชื่อ `Circle` ที่มี member variable `double radius` (private) พร้อม
   constructor ที่รับค่ารัศมี, member function `area()` และ `circumference()` (เส้นรอบวง)
   ที่เป็น `const` ทั้งคู่ และ method `setRadius(double newRadius)` ที่ตรวจสอบว่าค่าต้อง
   มากกว่า 0 เท่านั้น (ถ้าไม่ใช่ให้พิมพ์ข้อความเตือนแล้วไม่เปลี่ยนค่า)
2. เขียน class ชื่อ `Student` ที่มี member variable `std::string name` และ `double gpa`
   (private ทั้งคู่) พร้อม constructor, getter ทั้งสองตัว, และ method `isHonor()` ที่คืน
   `true` ถ้า `gpa >= 3.5`
3. เขียน class `Temperature` ที่เก็บอุณหภูมิเป็นองศาเซลเซียส (private) พร้อม method
   `toFahrenheit() const` ที่คำนวณและคืนค่าองศาฟาเรนไฮต์ (สูตร `F = C * 9/5 + 32`) โดยไม่
   แก้ไขค่าองศาเซลเซียสเดิม
4. แยก class `Circle` จากข้อ 1 ออกเป็น `Circle.h` และ `Circle.cpp` ตาม convention ที่เรียน
   ในหัวข้อ 45.6 แล้วเขียน `main` แยกอีกไฟล์เพื่อทดสอบ
5. เขียน class `Stopwatch` ที่มี member `int seconds` (private, เริ่มต้นที่ 0 ผ่าน
   constructor) พร้อม method `tick()` ที่เพิ่มค่า `seconds` ทีละ 1 และคืนค่าเป็น `Stopwatch&`
   (ใช้เทคนิค method chaining ด้วย `return *this;` เหมือนตัวอย่าง `Counter` ในหัวข้อ 45.5)
   แล้วทดลองเรียก `sw.tick().tick().tick();`
6. อธิบายด้วยคำพูดตัวเองว่าทำไม `BankAccount` ในหัวข้อ 45.8 ถึงไม่ควรเปลี่ยน `private` เป็น
   `public` สำหรับ member `balance` แม้จะทำให้เขียนโค้ดในฝั่งผู้ใช้สั้นลงก็ตาม

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>

class Circle {
private:
    double radius;

public:
    Circle(double radius) {
        this->radius = radius;
    }

    double area() const {
        return 3.14159265358979 * radius * radius;
    }

    double circumference() const {
        return 2 * 3.14159265358979 * radius;
    }

    void setRadius(double newRadius) {
        if (newRadius <= 0) {
            std::cout << "รัศมีต้องมากกว่า 0 เท่านั้น\n";
            return;
        }
        radius = newRadius;
    }

    double getRadius() const {
        return radius;
    }
};

int main() {
    Circle c(5.0);
    std::cout << "รัศมี " << c.getRadius() << ": area = " << c.area()
              << ", circumference = " << c.circumference() << '\n';

    c.setRadius(-2.0); // ควรถูกปฏิเสธ เพราะติดลบ
    std::cout << "หลังพยายามตั้งรัศมีติดลบ: รัศมียังเป็น " << c.getRadius() << '\n';

    c.setRadius(10.0);
    std::cout << "หลัง setRadius(10.0): area = " << c.area() << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 exercise1.cpp -o exercise1
./exercise1
```

```
รัศมี 5: area = 78.5398, circumference = 31.4159
รัศมีต้องมากกว่า 0 เท่านั้น
หลังพยายามตั้งรัศมีติดลบ: รัศมียังเป็น 5
หลัง setRadius(10.0): area = 314.159
```

### แนวทางเฉลยข้อ 5

```cpp
#include <iostream>

class Stopwatch {
private:
    int seconds;

public:
    Stopwatch() {
        this->seconds = 0;
    }

    Stopwatch& tick() {
        this->seconds += 1;
        return *this;
    }

    int getSeconds() const {
        return seconds;
    }
};

int main() {
    Stopwatch sw;
    sw.tick().tick().tick();
    std::cout << "หลังเรียก tick() 3 ครั้ง: seconds = " << sw.getSeconds() << '\n';

    sw.tick();
    std::cout << "หลังเรียก tick() อีก 1 ครั้ง: seconds = " << sw.getSeconds() << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 exercise5.cpp -o exercise5
./exercise5
```

```
หลังเรียก tick() 3 ครั้ง: seconds = 3
หลังเรียก tick() อีก 1 ครั้ง: seconds = 4
```

`tick()` คืนค่า `*this` (dereference `this` ให้เป็น `Stopwatch&` ที่ชี้กลับไปยัง object
เดิม) ทำให้เรียกต่อกันเป็นสาย `sw.tick().tick().tick();` ได้ในบรรทัดเดียว — เทคนิค Method
Chaining นี้จะกลับมาใช้บ่อยมากเมื่อออกแบบ Fluent Interface ในโค้ด C++ สมัยใหม่

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจแนวคิด **Encapsulation** ซึ่งเป็นหัวใจแรกของ OOP: การรวมข้อมูลและพฤติกรรมที่เกี่ยวข้อง
  ไว้เป็นหน่วยเดียวกันในรูปแบบ class
- แยกความแตกต่างระหว่าง **`class` และ `struct`** ได้อย่างชัดเจน: ต่างกันแค่ default access
  specifier เท่านั้น (`public` vs `private`)
- ประกาศและใช้งาน **member variable** และ **member function** ได้อย่างถูกต้อง พร้อมเข้าใจ
  ว่า `const` ท้าย member function มีความหมายอย่างไร
- สร้าง **object** จาก class ได้หลายตัว และเข้าใจว่าแต่ละ object มีสำเนา member variable
  เป็นของตัวเองแยกจากกัน
- เข้าใจและใช้งาน **`this` pointer** ได้อย่างถูกต้อง โดยเฉพาะการแก้ปัญหา shadowing เมื่อชื่อ
  พารามิเตอร์ชนกับ member variable และเทคนิค Method Chaining ด้วย `return *this;`
- แยก **class declaration (.h)** ออกจาก **implementation (.cpp)** ตาม convention มาตรฐาน
- เขียน class ที่ใช้งานได้จริงสองตัวอย่างครบวงจร: `Rectangle` และ `BankAccount`

ตอนนี้เรารู้จักส่วนประกอบพื้นฐานของ class ครบแล้ว แต่ยังมีคำถามสำคัญที่ยังไม่ได้ตอบ: object
`Rectangle r(4.0, 5.0);` ทำงานได้อย่างไรตอนสร้าง? ทำไมเราถึงเขียน `Rectangle::Rectangle(...)`
ได้? และเมื่อ object หมด scope ไป มีอะไรเกิดขึ้นกับหน่วยความจำที่มันถืออยู่บ้าง? คำถามเหล่านี้
จะได้คำตอบครบถ้วนใน **Part 46** ที่จะเจาะลึกเรื่อง **Constructor และ Destructor**

**ต่อไป:** [Part 46 — Constructor และ Destructor](./part-046-constructors-destructors.md)
