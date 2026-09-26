# Part 46: Constructor และ Destructor (Step 361–368)

> Module D — เริ่มต้น C++ และ OOP | Part 46 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 361–368
> Part ก่อนหน้า: [Part 45 — เริ่มต้น Class และ Object](./part-045-classes-objects.md) | Part ถัดไป: [Part 47 — Encapsulation และ Access Specifier](./part-047-encapsulation.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายและใช้งาน **Default Constructor** ได้อย่างถูกต้อง รวมถึงเข้าใจว่า compiler สร้าง
   default constructor ให้อัตโนมัติเมื่อไหร่ และหยุดสร้างให้เมื่อไหร่
2. เขียน **Parameterized Constructor** และใช้ **Constructor Overloading** เพื่อให้ class
   เดียวกันสร้าง object ได้หลายรูปแบบ
3. ใช้ **Member Initializer List** ได้อย่างถูกต้อง พร้อมอธิบายได้ว่าทำไมมันมีประสิทธิภาพ
   มากกว่าการ assign ค่าใน body ของ constructor และรู้กฎเรื่องลำดับการ initialize
4. แยกความแตกต่างระหว่าง **Copy Constructor** แบบที่ compiler สร้างให้ (default, shallow
   copy) กับแบบที่เขียนเอง (custom, deep copy) และรู้ว่าเมื่อไหร่ต้องเขียนเอง
5. อธิบายปัญหา **Shallow Copy** ที่นำไปสู่ Double Free ได้ และเขียน Copy Constructor แบบ
   **Deep Copy** เพื่อแก้ปัญหานี้
6. อธิบายได้ว่า **Destructor** ถูกเรียกเมื่อไหร่บ้าง (ออกจาก scope, `delete`, ลำดับการทำลาย
   ของหลาย object)
7. อธิบายและเขียนโค้ดตามแนวคิด **RAII (Resource Acquisition Is Initialization)** เบื้องต้น
   ด้วย class ที่จัดการ dynamic memory ด้วยตัวเอง

---

## 46.1 Default Constructor (Step 361)

**Constructor** คือ member function พิเศษที่ถูกเรียก **โดยอัตโนมัติ** ทุกครั้งที่มีการสร้าง
object ใหม่จาก class มีหน้าที่หลักคือ "เตรียมค่าเริ่มต้น" ให้ member variable ทุกตัวพร้อมใช้งาน
ก่อนที่ใครจะเรียกใช้ object นั้น

Constructor มีกฎรูปแบบ 2 ข้อที่ตายตัว: **ชื่อต้องตรงกับชื่อ class เป๊ะ** และ **ห้ามมี return
type ใดๆ เลย** (แม้แต่ `void` ก็ห้ามใส่)

**Default Constructor** คือ constructor ที่ **ไม่รับพารามิเตอร์เลย** (หรือพารามิเตอร์ทุกตัวมี
default argument ครบ ตามที่เรียนใน Part 44) ถูกเรียกเมื่อสร้าง object โดยไม่ส่ง argument ใดๆ

```cpp
// default_ctor.cpp
#include <iostream>
#include <string>

class Book {
private:
    std::string title;
    int pages;

public:
    Book() {
        title = "ไม่ระบุชื่อ";
        pages = 0;
        std::cout << "เรียก default constructor สร้าง Book\n";
    }

    void printInfo() const {
        std::cout << "ชื่อหนังสือ: " << title << ", จำนวนหน้า: " << pages << '\n';
    }
};

int main() {
    Book b1; // ไม่ส่ง argument ใดๆ เลย -> เรียก default constructor อัตโนมัติ
    b1.printInfo();
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 default_ctor.cpp -o default_ctor
./default_ctor
```

```
เรียก default constructor สร้าง Book
ชื่อหนังสือ: ไม่ระบุชื่อ, จำนวนหน้า: 0
```

### compiler สร้าง default constructor ให้อัตโนมัติเมื่อไหร่

จุดสำคัญที่ต้องเข้าใจ: **ถ้า class ไม่มี constructor ใดๆ เขียนไว้เลยแม้แต่ตัวเดียว** compiler
จะสร้าง default constructor ให้อัตโนมัติแบบ "ว่างเปล่า" (ไม่ทำอะไรเลยกับ member ที่เป็น
built-in type อย่าง `int`/`double` — ค่าจะเป็นขยะเหมือนตัวแปร local ที่ไม่ initialize ตามที่
เรียนใน Part 1 แต่ member ที่เป็น class type อย่าง `std::string` จะถูกเรียก default
constructor ของมันเองให้)

แต่ **ทันทีที่เราเขียน constructor แบบมีพารามิเตอร์เองสักตัวเดียว compiler จะหยุดสร้าง
default constructor ให้ทันที** ลองดูตัวอย่างต่อไปนี้:

```cpp
// no_default.cpp — ตัวอย่างนี้ "ผิดโดยตั้งใจ" เพื่อสาธิตว่า compiler ไม่สร้าง default
// constructor ให้ ทันทีที่เราเขียน constructor แบบมีพารามิเตอร์เองอย่างน้อยหนึ่งตัว
#include <string>

class Book {
private:
    std::string title;
    int pages;

public:
    Book(const std::string& title, int pages) : title(title), pages(pages) {}
};

int main() {
    Book b; // ERROR: ไม่มี default constructor ให้ใช้อีกต่อไป
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 no_default.cpp -o no_default
```

```
error: no matching function for call to 'Book::Book()'
note: candidate: 'Book::Book(const std::string&, int)'
note:   candidate expects 2 arguments, 0 provided
```

ถ้าต้องการให้ `Book b;` ยังใช้งานได้อยู่ ต้องเขียน default constructor เพิ่มเข้าไปเองอย่าง
ชัดเจน (หรือใช้ `Book() = default;` เพื่อขอให้ compiler สร้างแบบเดิมให้กลับมา ซึ่งเป็น syntax
ของ C++11 ที่จะพบบ่อยในโค้ดสมัยใหม่)

---

## 46.2 Parameterized Constructor และ Constructor Overloading (Step 362)

**Parameterized Constructor** คือ constructor ที่รับพารามิเตอร์ เพื่อกำหนดค่าเริ่มต้นให้
object ตั้งแต่ตอนสร้างตามที่ผู้เรียกต้องการ และเนื่องจาก constructor ก็คือ member function
ชนิดหนึ่ง มันจึงใช้กฎ **Function Overloading** จาก Part 44 ได้เหมือนกันทุกประการ — เขียน
constructor ชื่อเดียวกัน (เพราะบังคับต้องตรงกับชื่อ class) แต่รับพารามิเตอร์ต่างกันได้หลายแบบ
เรียกว่า **Constructor Overloading**

```cpp
// ctor_overload.cpp
#include <iostream>
#include <string>

class Book {
private:
    std::string title;
    int pages;

public:
    // Default constructor: ใช้ตอนไม่ส่ง argument ใดๆ เลย
    Book() : title("ไม่ระบุชื่อ"), pages(0) {
        std::cout << "เรียก Book() default constructor\n";
    }

    // Parameterized constructor: รับแค่ title
    Book(const std::string& title) : title(title), pages(0) {
        std::cout << "เรียก Book(title) constructor\n";
    }

    // Parameterized constructor: รับทั้ง title และ pages (constructor overloading)
    Book(const std::string& title, int pages) : title(title), pages(pages) {
        std::cout << "เรียก Book(title, pages) constructor\n";
    }

    void printInfo() const {
        std::cout << "ชื่อหนังสือ: " << title << ", จำนวนหน้า: " << pages << '\n';
    }
};

int main() {
    Book b1;                          // เรียก Book()
    Book b2("C++ เบื้องต้น");          // เรียก Book(title)
    Book b3("C++ ขั้นสูง", 350);       // เรียก Book(title, pages)

    b1.printInfo();
    b2.printInfo();
    b3.printInfo();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ctor_overload.cpp -o ctor_overload
./ctor_overload
```

```
เรียก Book() default constructor
เรียก Book(title) constructor
เรียก Book(title, pages) constructor
ชื่อหนังสือ: ไม่ระบุชื่อ, จำนวนหน้า: 0
ชื่อหนังสือ: C++ เบื้องต้น, จำนวนหน้า: 0
ชื่อหนังสือ: C++ ขั้นสูง, จำนวนหน้า: 350
```

compiler เลือก constructor ที่ตรงกับจำนวนและชนิด argument ที่ส่งเข้าไปให้เองโดยอัตโนมัติ
ด้วยกลไก Overload Resolution แบบเดียวกับที่เรียนใน Part 44 ทุกประการ — เพียงแต่ครั้งนี้
"ฟังก์ชัน" ที่ overload คือ constructor ของ class นั่นเอง

> **สังเกต syntax `Book b2("C++ เบื้องต้น");`** — นี่คือการเรียก constructor แบบ **Direct
> Initialization** (ไม่มี `=` คั่นกลาง) ซึ่งเป็นรูปแบบที่แนะนำเมื่อสร้าง object ด้วย
> parameterized constructor เพราะชัดเจนและมีประสิทธิภาพเท่ากับหรือดีกว่ารูปแบบอื่นเสมอ

---

## 46.3 Member Initializer List: Syntax และเหตุผลด้านประสิทธิภาพ (Step 363)

จนถึงตอนนี้เราใช้ syntax `: title(title), pages(pages)` ต่อท้าย parameter list ของ
constructor มาตลอดโดยยังไม่อธิบายละเอียด — นี่คือ **Member Initializer List** วิธีการกำหนด
ค่าเริ่มต้นให้ member variable ที่ "ถูกต้อง" ที่สุดในภาษา C++

รูปแบบทั่วไป:

```cpp
ClassName(พารามิเตอร์) : member1(ค่า1), member2(ค่า2), ... {
    // body ของ constructor (โค้ดที่เหลือ ถ้ามี)
}
```

### ทำไม Initializer List ถึงมีประสิทธิภาพกว่าการ assign ใน body

หลายคนเขียน constructor แบบนี้แทน (assign ค่าใน body แทนที่จะใช้ initializer list):

```cpp
class Bad {
    std::string name;
public:
    Bad(const std::string& n) {
        name = n; // assignment ใน body
    }
};
```

โค้ดแบบนี้ **ทำงานถูกต้อง** แต่ **เปลืองกว่าที่ควรจะเป็น** เพราะเบื้องหลังจริงๆ แล้วมี 2
ขั้นตอนเกิดขึ้น: (1) `name` ถูก **default-construct** ก่อน (สร้าง `std::string` ว่างเปล่าขึ้นมา)
แล้ว (2) ค่อยถูก **assign** ค่า `n` ทับเข้าไปทีหลังใน body — ทำงานซ้ำซ้อนโดยไม่จำเป็น

ในขณะที่ initializer list `Bad(const std::string& n) : name(n) {}` จะ **construct `name`
ด้วยค่า `n` ไปเลยในขั้นตอนเดียว** ไม่มีการสร้างค่าว่างขึ้นมาก่อนแล้วทิ้งทันที

มาดูของจริงด้วยการติดตาม (trace) การเรียก constructor/`operator=`:

```cpp
// init_list_efficiency.cpp
#include <iostream>
#include <string>

class Traceable {
private:
    std::string data;

public:
    Traceable() {
        std::cout << "  [Traceable] default constructor ทำงาน\n";
    }

    Traceable(const std::string& s) : data(s) {
        std::cout << "  [Traceable] constructor รับค่า \"" << s << "\" ทำงาน\n";
    }

    Traceable& operator=(const std::string& s) {
        std::cout << "  [Traceable] operator= รับค่า \"" << s << "\" ทำงาน\n";
        data = s;
        return *this;
    }
};

class BadDesign {
private:
    Traceable t;

public:
    BadDesign(const std::string& s) {
        t = s; // ทำงาน 2 รอบ: default construct t ก่อน แล้วค่อย assign ทีหลัง
    }
};

class GoodDesign {
private:
    Traceable t;

public:
    GoodDesign(const std::string& s) : t(s) {
        // ทำงานรอบเดียว: สร้าง t ด้วยค่า s ไปเลยผ่าน member initializer list
    }
};

int main() {
    std::cout << "สร้าง BadDesign:\n";
    BadDesign bad("ข้อมูล A");

    std::cout << "สร้าง GoodDesign:\n";
    GoodDesign good("ข้อมูล B");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 init_list_efficiency.cpp -o init_list_efficiency
./init_list_efficiency
```

```
สร้าง BadDesign:
  [Traceable] default constructor ทำงาน
  [Traceable] operator= รับค่า "ข้อมูล A" ทำงาน
สร้าง GoodDesign:
  [Traceable] constructor รับค่า "ข้อมูล B" ทำงาน
```

เห็นได้ชัดเจน: `BadDesign` เรียกทั้ง default constructor และ `operator=` รวม 2 ครั้ง ในขณะที่
`GoodDesign` เรียก constructor ที่รับค่าตรงๆ แค่ **ครั้งเดียว** — สำหรับ `std::string` สั้นๆ
ความต่างนี้อาจดูเล็กน้อย แต่เมื่อ member เป็น object ขนาดใหญ่ (เช่น `std::vector` ที่มีข้อมูล
เป็นล้านตัว) หรือ constructor ถูกเรียกซ้ำๆ นับล้านครั้ง ความต่างนี้จะกลายเป็นปัญหาด้าน
ประสิทธิภาพที่ชัดเจนมาก

### เหตุผลที่สอง (สำคัญกว่า): member บางชนิด "ต้อง" ใช้ initializer list เท่านั้น

นอกจากเรื่องประสิทธิภาพ ยังมี member บางประเภทที่ **ไม่มีทางกำหนดค่าผ่านการ assign ใน body
ได้เลย** เช่น **`const` member** และ **reference member** — เพราะทั้งสองแบบต้องมีค่าที่
แน่นอนตั้งแต่ตอน "เกิด" เท่านั้น จะมาแก้ไขทีหลังไม่ได้ (ตรงตามธรรมชาติของ `const` ที่เรียน
มาตั้งแต่ Part 43)

```cpp
// const_member.cpp
#include <iostream>

class Config {
private:
    const int id; // const member: กำหนดค่าได้ครั้งเดียวตอนสร้างเท่านั้น

public:
    Config(int idValue) : id(idValue) { // ต้องใช้ initializer list เท่านั้น
        // ห้ามเขียน id = idValue; ในนี้ เพราะ id เป็น const แก้ไขทีหลังไม่ได้
    }

    int getId() const {
        return id;
    }
};

int main() {
    Config c(42);
    std::cout << "config id = " << c.getId() << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 const_member.cpp -o const_member
./const_member
```

```
config id = 42
```

ถ้าลองย้ายการกำหนดค่าไปไว้ใน body แทน จะเกิด compile error ทันที:

```cpp
// const_member_bad.cpp — ตัวอย่างนี้ "ผิดโดยตั้งใจ" เพื่อสาธิตว่า const member
// แก้ไขใน body ของ constructor ไม่ได้
class Config {
private:
    const int id;

public:
    Config(int idValue) {
        id = idValue; // ERROR: assignment of read-only member 'Config::id'
    }
};
```

```
error: uninitialized const member in 'const int'
error: assignment of read-only member 'Config::id'
```

> **สรุปกฎทอง**: ให้ใช้ **Member Initializer List เสมอ** สำหรับทุก member ที่กำหนดค่าได้
> ตั้งแต่ตอนสร้าง แม้จะไม่ใช่ `const`/reference ก็ตาม เพราะได้ทั้งประสิทธิภาพที่ดีกว่าและ
> ความสม่ำเสมอของโค้ด เก็บการเขียนโค้ดใน body ของ constructor ไว้สำหรับ logic ที่ซับซ้อนกว่า
> การกำหนดค่าเริ่มต้นตรงๆ เท่านั้น (เช่น การตรวจสอบความถูกต้องของ argument หรือการพิมพ์ log)

---

## 46.4 กฎการเรียงลำดับ: Initialize ตามลำดับการประกาศ ไม่ใช่ลำดับใน Initializer List (Step 364)

กฎที่มือใหม่เข้าใจผิดบ่อยที่สุดเรื่อง initializer list คือ: **compiler จะ initialize member
ตามลำดับที่ member ถูก "ประกาศ" ไว้ใน class เท่านั้น ไม่ใช่ตามลำดับที่เราเขียนไว้ใน
initializer list** แม้ทั้งสองลำดับจะดูสมเหตุสมผลพอกัน แต่ถ้าเขียนไม่ตรงกัน อาจนำไปสู่บั๊กที่
วินิจฉัยยากมาก โดยเฉพาะเมื่อ member ตัวหนึ่งต้องพึ่งค่าของ member อีกตัวหนึ่ง

```cpp
// reorder_bug.cpp — ตัวอย่างนี้ "จงใจ" เขียนบั๊กเรื่องลำดับ initializer list เพื่อสาธิตปัญหา
#include <iostream>

class Bad {
private:
    int b; // ประกาศ b ก่อน a
    int a;

public:
    // ตั้งใจเขียนใน initializer list ตามลำดับ a ก่อน b แต่ compiler ไม่สนลำดับที่เขียน
    // มันจะ initialize ตามลำดับ "การประกาศ member" เสมอ คือ b ก่อน แล้วค่อย a
    Bad(int value) : a(value), b(a) {
        // เจตนาคืออยากให้ b = a = value แต่เพราะ b ถูก initialize ก่อน a (ตามลำดับประกาศ)
        // ตอน initialize b, a ยังไม่ถูกกำหนดค่าเลย -> b ได้ค่าขยะ (Undefined Behavior)
    }

    void print() const {
        std::cout << "a = " << a << ", b = " << b << '\n';
    }
};

int main() {
    Bad obj(99);
    obj.print(); // b อาจไม่ใช่ 99 เลย เพราะบั๊กเรื่องลำดับ initialize
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 reorder_bug.cpp -o reorder_bug
```

compiler เตือนปัญหานี้ให้ทันทีด้วย `-Wreorder` (รวมอยู่ใน `-Wall` แล้ว) และยังเตือนเรื่องการ
ใช้ค่าที่ยังไม่ถูก initialize ด้วย:

```
warning: 'Bad::a' will be initialized after [-Wreorder]
warning:   'int Bad::b' [-Wreorder]
warning:   when initialized here [-Wreorder]
warning: member 'Bad::a' is used uninitialized [-Wuninitialized]
```

รันดูผลลัพธ์จริง (ค่า `b` อาจต่างกันไปในแต่ละเครื่อง เพราะเป็นค่าขยะจาก Undefined Behavior):

```
a = 99, b = 0
```

`b` ไม่ได้เป็น `99` ตามที่ตั้งใจเลย เพราะตอน compiler ประมวลผล `b(a)` (initialize `b` ด้วยค่า
ของ `a`) นั้น `a` **ยังไม่ถูก initialize เลย** (เพราะ `a` ถูกประกาศไว้ **หลัง** `b` ใน class)
ค่าที่ `b` ได้รับจึงเป็นขยะจากหน่วยความจำที่ยังไม่ถูกกำหนดค่า

**วิธีแก้ที่ถูกต้องจริงๆ มีสองทาง**: (1) สลับลำดับการ**ประกาศ** member ให้ `a` มาก่อน `b`
เพื่อให้ตรงกับ logic ที่ต้องการ หรือ (2) เขียน initializer list ให้ **ตรงกับลำดับการประกาศ
เสมอ** (ซึ่งเป็นวิธีที่ปลอดภัยกว่าและควรทำเป็นนิสัย)

> **กฎทองของหลักสูตรนี้**: **เขียน initializer list ให้เรียงลำดับตรงกับลำดับการประกาศ
> member ใน class เสมอ** แม้ compiler จะไม่บังคับ (แค่เตือนด้วย `-Wreorder`) แต่การเขียนให้
> ตรงกันช่วยป้องกันบั๊กประเภทนี้ได้เด็ดขาด และทำให้โค้ดอ่านง่ายขึ้นด้วย เพราะลำดับที่เขียนตรง
> กับลำดับที่ compiler จะทำงานจริง

---

## 46.5 Copy Constructor: Default (Shallow Copy) (Step 365)

**Copy Constructor** คือ constructor พิเศษที่ถูกเรียกเมื่อสร้าง object ใหม่ **จากการคัดลอก
object ที่มีอยู่แล้ว** ของ class เดียวกัน มี signature ที่ตายตัวคือ `ClassName(const
ClassName& other)`

เหตุการณ์ที่เรียก copy constructor มีหลายแบบ:

```cpp
Book b1("C++ เบื้องต้น", 300);
Book b2 = b1;   // เรียก copy constructor (แม้จะมี = ก็ตาม เพราะเป็นการ "สร้าง" object ใหม่)
Book b3(b1);    // เรียก copy constructor แบบ direct initialization
```

เช่นเดียวกับ default constructor, **ถ้า class ไม่ได้เขียน copy constructor เองเลย** compiler
จะสร้างให้อัตโนมัติ โดยพฤติกรรม default คือ **คัดลอก member variable ทีละตัวแบบตรงๆ**
(เรียกว่า **Member-wise Copy** หรือ **Shallow Copy**) — สำหรับ member ที่เป็น type ธรรมดา
(`int`, `double`) หรือ class ที่จัดการตัวเองได้ดีอยู่แล้ว (`std::string`, `std::vector`)
พฤติกรรมนี้ถูกต้องสมบูรณ์แบบ ไม่มีปัญหาอะไรเลย

แต่ถ้า class มี **raw pointer ที่เป็นเจ้าของหน่วยความจำเอง** (เช่นได้มาจาก `new`) Shallow
Copy จะกลายเป็นปัญหาใหญ่ทันที เพราะมันจะ **คัดลอกค่า pointer ตรงๆ** โดยไม่คัดลอกข้อมูลที่
pointer ชี้ไปให้ใหม่เลย ผลคือ object 2 ตัว (`a` กับ `b`) จะ **ชี้ไปที่หน่วยความจำก้อนเดียวกัน**

---

## 46.6 Shallow Copy vs Deep Copy: เมื่อไหร่ต้องเขียน Copy Constructor เอง (Step 366)

มาดูปัญหาของ Shallow Copy แบบที่เกิดขึ้นจริง ด้วย class `IntBuffer` ที่ถือ `int*` ที่ได้จาก
`new[]`:

```cpp
// shallow_copy_bug.cpp — ตัวอย่างนี้ "จงใจ" เขียนบั๊ก shallow copy เพื่อสาธิตปัญหา double
// free คำเตือน: โปรแกรมนี้จะ crash ตอนรันจริง ห้ามใช้ pattern นี้ในโค้ดจริงเด็ดขาด
#include <iostream>

class IntBuffer {
private:
    int* data;
    int size;

public:
    IntBuffer(int size) : data(new int[static_cast<std::size_t>(size)]), size(size) {
        for (int i = 0; i < size; ++i) {
            data[i] = i;
        }
    }

    // ไม่ได้เขียน copy constructor เอง -> compiler สร้าง "shallow copy" ให้อัตโนมัติ
    // (คัดลอกค่า pointer data ตรงๆ ไม่ได้คัดลอกข้อมูลที่ pointer ชี้ไปให้ใหม่)

    ~IntBuffer() {
        std::cout << "  ~IntBuffer() กำลัง delete[] data ที่ address " << data << '\n';
        delete[] data;
    }

    void print() const {
        std::cout << "  data address = " << data << ", data[0] = " << data[0] << '\n';
    }
};

int main() {
    {
        IntBuffer a(3);
        std::cout << "a: "; a.print();

        IntBuffer b = a; // เรียก compiler-generated copy constructor -> shallow copy!
        std::cout << "b (สำเนาจาก a): "; b.print();
        std::cout << "สังเกตว่า address ของ a และ b ชี้ไปที่เดียวกัน!\n";

        // เมื่อออกจาก scope นี้ ทั้ง a และ b จะถูก destroy
        // ทั้งคู่จะพยายาม delete[] pointer เดียวกัน -> double free (Undefined Behavior)
    }
    std::cout << "จบโปรแกรม\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 shallow_copy_bug.cpp -o shallow_copy_bug
./shallow_copy_bug
```

โค้ดชุดนี้ **compile ผ่านโดยไม่มี warning แม้แต่ตัวเดียว** (เพราะในเชิงไวยากรณ์ไม่มีอะไรผิดเลย)
แต่พอรันจริงจะพังทันที:

```
a:   data address = 0x... , data[0] = 0
b (สำเนาจาก a):   data address = 0x... , data[0] = 0
สังเกตว่า address ของ a และ b ชี้ไปที่เดียวกัน!
  ~IntBuffer() กำลัง delete[] data ที่ address 0x...
free(): double free detected in tcache 2
Aborted
```

ทันทีที่ scope `{ ... }` จบลง `b` ถูกทำลายก่อน (ตามลำดับ LIFO ที่จะอธิบายในหัวข้อ 46.7)
เรียก `delete[] data` ไปครั้งหนึ่ง แล้วพอถึงคิว `a` ถูกทำลาย มันก็เรียก `delete[] data` **ที่
เดียวกันซ้ำอีกครั้ง** — นี่คือ **Double Free**: การพยายามคืนหน่วยความจำก้อนเดียวกันสองครั้ง
ซึ่งเป็น Undefined Behavior ที่ระบบส่วนใหญ่จะตรวจจับได้และ crash โปรแกรมทันที (บางระบบอาจไม่
crash ทันทีแต่ทำให้หน่วยความจำเสียหายแบบเงียบๆ ซึ่งอันตรายยิ่งกว่า)

### แก้ปัญหาด้วย Deep Copy Constructor ที่เขียนเอง

วิธีแก้คือเขียน **Copy Constructor เอง** ให้ **จัดสรรหน่วยความจำก้อนใหม่** แล้วคัดลอก
**ข้อมูล** (ไม่ใช่แค่ค่า pointer) ไปทีละตัว เรียกว่า **Deep Copy**:

```cpp
// deep_copy_fix.cpp
#include <iostream>

class IntBuffer {
private:
    int* data;
    int size;

public:
    IntBuffer(int size) : data(new int[static_cast<std::size_t>(size)]), size(size) {
        for (int i = 0; i < size; ++i) {
            data[i] = i;
        }
    }

    // Copy constructor แบบ deep copy ที่เขียนเอง: จัดสรรหน่วยความจำใหม่ แล้วคัดลอกค่าไปทีละตัว
    IntBuffer(const IntBuffer& other)
        : data(new int[static_cast<std::size_t>(other.size)]), size(other.size) {
        for (int i = 0; i < size; ++i) {
            data[i] = other.data[i];
        }
        std::cout << "  เรียก deep copy constructor: จัดสรร memory ใหม่ที่ " << data << '\n';
    }

    ~IntBuffer() {
        std::cout << "  ~IntBuffer() กำลัง delete[] data ที่ address " << data << '\n';
        delete[] data;
    }

    void print() const {
        std::cout << "  data address = " << data << ", data[0] = " << data[0] << '\n';
    }
};

int main() {
    {
        IntBuffer a(3);
        std::cout << "a: "; a.print();

        IntBuffer b = a; // เรียก custom deep copy constructor -> คนละ memory กับ a
        std::cout << "b (สำเนาจาก a): "; b.print();
        std::cout << "สังเกตว่า address ของ a และ b ต่างกัน (deep copy สำเร็จ)\n";
    }
    std::cout << "จบโปรแกรมปกติ ไม่มี double free\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 deep_copy_fix.cpp -o deep_copy_fix
./deep_copy_fix
```

```
a:   data address = 0x564ac12862b0, data[0] = 0
  เรียก deep copy constructor: จัดสรร memory ใหม่ที่ 0x564ac12872e0
b (สำเนาจาก a):   data address = 0x564ac12872e0, data[0] = 0
สังเกตว่า address ของ a และ b ต่างกัน (deep copy สำเร็จ)
  ~IntBuffer() กำลัง delete[] data ที่ address 0x564ac12872e0
  ~IntBuffer() กำลัง delete[] data ที่ address 0x564ac12862b0
```

ครั้งนี้ `a` และ `b` มี address ของ `data` **ต่างกันอย่างชัดเจน** — แก้ไขค่าใน `b` จะไม่มีผล
ต่อ `a` เลย และเมื่อถึงเวลาทำลาย object ทั้งคู่ก็ `delete[]` คนละก้อนหน่วยความจำ ไม่ชนกัน
ตรวจสอบด้วย valgrind ยืนยันว่าไม่มี memory leak หรือ error ใดๆ เลย

> **กฎทองสำหรับการเลือกว่าต้องเขียน Copy Constructor เองหรือไม่**: ถ้า class มี member ที่
> เป็น **raw pointer ที่ class เป็นเจ้าของหน่วยความจำเอง** (ได้มาจาก `new`/`new[]` ภายใน
> constructor ของ class เอง) **ต้องเขียน copy constructor แบบ deep copy เองเสมอ** ถ้า member
> ทุกตัวเป็น type ที่จัดการตัวเองอยู่แล้ว (`int`, `double`, `std::string`, `std::vector`,
> smart pointer ที่จะเรียนใน Part 67) **ปล่อยให้ compiler สร้างให้ก็เพียงพอ** เพราะ type
> เหล่านั้นมี copy constructor ของตัวเองที่ทำ deep copy ถูกต้องอยู่แล้ว
>
> กฎนี้เป็นส่วนหนึ่งของแนวคิดที่เรียกว่า **Rule of Three** (ถ้า class ต้องเขียน destructor
> เองเพราะจัดการทรัพยากรเอง มักต้องเขียน copy constructor และ copy assignment operator เอง
> ด้วยเช่นกัน) ซึ่งจะเจาะลึกเต็มรูปแบบพร้อม **Rule of Five** อีกครั้งเมื่อเรียนเรื่อง Move
> Semantics ใน **Part 70**

---

## 46.7 Destructor และเมื่อไหร่ถูกเรียก (Step 367)

**Destructor** คือ member function พิเศษอีกตัวที่ถูกเรียก **โดยอัตโนมัติ** ก่อนที่ object
จะถูกทำลาย มีหน้าที่ตรงข้ามกับ constructor: "คืนทรัพยากร" ที่ object ถือครองอยู่ (หน่วยความจำ,
ไฟล์ที่เปิดค้างไว้, การเชื่อมต่อเครือข่าย ฯลฯ) ก่อนที่ object จะหายไปจากระบบ

รูปแบบ: ชื่อ destructor คือชื่อ class นำหน้าด้วย `~` (tilde), **ไม่รับพารามิเตอร์ใดๆ เลย**
และ **มีได้แค่ตัวเดียวต่อ class** (overload ไม่ได้ ต่างจาก constructor)

```cpp
class IntBuffer {
public:
    ~IntBuffer() {
        delete[] data;
    }
    // ...
};
```

### Destructor ถูกเรียกเมื่อไหร่บ้าง

```cpp
// destructor_timing.cpp
#include <iostream>
#include <string>

class Tracer {
private:
    std::string name;

public:
    Tracer(const std::string& name) : name(name) {
        std::cout << "สร้าง " << name << '\n';
    }

    ~Tracer() {
        std::cout << "ทำลาย " << name << '\n';
    }
};

void demo_scope() {
    std::cout << "-- เข้า demo_scope --\n";
    Tracer local("local (stack)");
    std::cout << "-- กำลังจะออกจาก demo_scope --\n";
} // local ถูก destroy ที่นี่ตอนออกจาก scope โดยอัตโนมัติ

int main() {
    std::cout << "=== ตัวอย่างที่ 1: destructor ทำงานตอนออกจาก scope ===\n";
    demo_scope();

    std::cout << "\n=== ตัวอย่างที่ 2: destructor ทำงานเมื่อเรียก delete ===\n";
    Tracer* heapObj = new Tracer("heap object");
    std::cout << "ยังไม่ delete...\n";
    delete heapObj; // destructor ถูกเรียกตรงนี้ทันที (ไม่ใช่ตอน program จบ)

    std::cout << "\n=== ตัวอย่างที่ 3: ลำดับการทำลาย object หลายตัวใน scope เดียวกัน ===\n";
    {
        Tracer first("first");
        Tracer second("second");
        Tracer third("third");
        std::cout << "-- กำลังจะออกจาก block --\n";
    } // ทำลายย้อนลำดับ: third, second, first (Last In First Out เหมือน stack)

    std::cout << "\nจบ main\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 destructor_timing.cpp -o destructor_timing
./destructor_timing
```

```
=== ตัวอย่างที่ 1: destructor ทำงานตอนออกจาก scope ===
-- เข้า demo_scope --
สร้าง local (stack)
-- กำลังจะออกจาก demo_scope --
ทำลาย local (stack)

=== ตัวอย่างที่ 2: destructor ทำงานเมื่อเรียก delete ===
สร้าง heap object
ยังไม่ delete...
ทำลาย heap object

=== ตัวอย่างที่ 3: ลำดับการทำลาย object หลายตัวใน scope เดียวกัน ===
สร้าง first
สร้าง second
สร้าง third
-- กำลังจะออกจาก block --
ทำลาย third
ทำลาย second
ทำลาย first

จบ main
```

สรุปกฎการเรียก destructor 3 สถานการณ์หลักจากตัวอย่างข้างต้น:

| สถานการณ์ | ตอนไหนที่ destructor ถูกเรียก |
|---|---|
| Object บน stack (local variable) | ทันทีที่ออกจาก scope (`{ }`) ที่ object นั้นถูกประกาศไว้ |
| Object บน heap (สร้างด้วย `new`) | ทันทีที่เรียก `delete` กับ pointer นั้น (ถ้าไม่เรียก `delete` เลย จะไม่ถูกทำลายเลย ⇒ **memory leak**) |
| หลาย object ใน scope เดียวกัน | ทำลายย้อนลำดับการสร้าง (**Last In, First Out** เหมือนโครงสร้างข้อมูล Stack ที่เรียนใน Part 20) |

> **ข้อควรระวังสำคัญ**: object ที่สร้างด้วย `new` แล้ว **ไม่เรียก `delete`** จะไม่ถูกทำลายเลย
> ตราบใดที่โปรแกรมยังทำงานอยู่ — destructor จะไม่ถูกเรียก หน่วยความจำ (และทรัพยากรอื่นที่
> destructor ควรจะคืน) จะรั่วไหลตลอดไป นี่คือ **Memory Leak** แบบเดียวกับที่เรียนใน Part 11
> (Dynamic Memory) และ Part 38 (Valgrind) เพียงแต่คราวนี้เกิดกับ object แทนที่จะเป็น raw
> `malloc`/`free`

---

## 46.8 RAII เบื้องต้น: Constructor Acquire, Destructor Release (Step 368)

จากทุกอย่างที่เรียนมาใน Part นี้ เราสามารถสรุปเป็นแนวคิดออกแบบที่สำคัญที่สุดอย่างหนึ่งของ
C++ ได้ นั่นคือ **RAII (Resource Acquisition Is Initialization)**:

> **แนวคิด RAII**: ผูก "การจัดหาทรัพยากร (Resource Acquisition)" ไว้กับ **constructor**
> และผูก "การคืนทรัพยากร (Resource Release)" ไว้กับ **destructor** ของ object เดียวกันเสมอ
> ทำให้วงจรชีวิตของทรัพยากร (หน่วยความจำ, ไฟล์, การเชื่อมต่อเครือข่าย, mutex lock ฯลฯ) ผูก
> ติดกับวงจรชีวิตของ object โดยอัตโนมัติ — **ไม่มีทางลืมคืนทรัพยากรได้เลย** เพราะ compiler
> รับประกันว่า destructor จะถูกเรียกเสมอเมื่อ object หมดอายุ ไม่ว่าจะออกจาก scope ตามปกติ
> หรือมี exception เกิดขึ้นระหว่างทางก็ตาม (จะเจาะลึกเรื่อง exception ใน Part 54)

ลองรวมทุกอย่างที่เรียนมา (constructor, member initializer list, deep copy, destructor) เข้า
เป็น class เดียวที่ปฏิบัติตามแนวคิด RAII อย่างสมบูรณ์:

```cpp
// raii_demo.cpp
#include <iostream>
#include <stdexcept>

// SafeIntArray สาธิตแนวคิด RAII (Resource Acquisition Is Initialization):
// - Constructor "จัดหา (acquire)" ทรัพยากร (หน่วยความจำ) ให้เสร็จสรรพตั้งแต่ตอนสร้าง object
// - Destructor "คืน (release)" ทรัพยากรนั้นให้อัตโนมัติ ไม่ว่า object จะหมด scope แบบไหนก็ตาม
// ผลคือผู้ใช้ class ไม่มีทาง "ลืม" คืนหน่วยความจำเลย เพราะมันผูกกับวงจรชีวิตของ object โดยตรง
class SafeIntArray {
private:
    int* data;
    int size;

public:
    explicit SafeIntArray(int size) : data(new int[static_cast<std::size_t>(size)]), size(size) {
        for (int i = 0; i < size; ++i) {
            data[i] = 0;
        }
        std::cout << "[acquire] จัดสรร memory สำหรับ " << size << " int ที่ address " << data << '\n';
    }

    // Deep copy constructor: ทุก object เป็นเจ้าของหน่วยความจำของตัวเองเสมอ
    SafeIntArray(const SafeIntArray& other)
        : data(new int[static_cast<std::size_t>(other.size)]), size(other.size) {
        for (int i = 0; i < size; ++i) {
            data[i] = other.data[i];
        }
        std::cout << "[acquire] deep copy สร้าง memory ใหม่ที่ address " << data << '\n';
    }

    ~SafeIntArray() {
        std::cout << "[release] คืน memory ที่ address " << data << '\n';
        delete[] data;
    }

    int get(int index) const {
        if (index < 0 || index >= size) {
            throw std::out_of_range("index อยู่นอกขอบเขตของ array");
        }
        return data[index];
    }

    void set(int index, int value) {
        if (index < 0 || index >= size) {
            throw std::out_of_range("index อยู่นอกขอบเขตของ array");
        }
        data[index] = value;
    }

    int getSize() const {
        return size;
    }
};

void use_array() {
    SafeIntArray arr(5); // acquire ตรงนี้
    for (int i = 0; i < arr.getSize(); ++i) {
        arr.set(i, i * 10);
    }

    std::cout << "ค่าภายใน arr: ";
    for (int i = 0; i < arr.getSize(); ++i) {
        std::cout << arr.get(i) << ' ';
    }
    std::cout << '\n';

    // ไม่ต้องเรียก delete[] เอง! ไม่ว่า use_array() จะ return ตามปกติ
    // หรือมี exception เกิดขึ้นระหว่างทาง destructor ของ arr ก็จะถูกเรียกอัตโนมัติเสมอ
} // release เกิดขึ้นอัตโนมัติตรงนี้ ไม่ว่าจะออกจากฟังก์ชันด้วยเหตุผลใดก็ตาม

int main() {
    use_array();
    std::cout << "กลับมาที่ main เรียบร้อย ไม่มี memory leak เกิดขึ้นเลย\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 raii_demo.cpp -o raii_demo
./raii_demo
```

```
[acquire] จัดสรร memory สำหรับ 5 int ที่ address 0x55d8b2f1c2b0
ค่าภายใน arr: 0 10 20 30 40
[release] คืน memory ที่ address 0x55d8b2f1c2b0
กลับมาที่ main เรียบร้อย ไม่มี memory leak เกิดขึ้นเลย
```

ตรวจสอบด้วย `valgrind` (จะเรียนละเอียดใน Part 38) ยืนยันว่าไม่มี memory leak หลงเหลือเลย:

```bash
valgrind --leak-check=full ./raii_demo
```

```
==...== HEAP SUMMARY:
==...==     in use at exit: 0 bytes in 0 blocks
==...==   total heap usage: 1 allocs, 1 frees, ...
==...== All heap blocks were freed -- no leaks are possible
```

### RAII แข็งแกร่งกว่าการเขียน "อย่าลืมคืนทรัพยากร" ด้วยตัวเองแค่ไหน

ลองดูสถานการณ์ที่ฟังก์ชันต้อง return กลางทางเพราะ error หรือเกิด exception — ถ้าใช้ raw
pointer ธรรมดา ผู้เขียนโค้ดต้อง "จำ" ที่จะ `delete` ก่อน return ทุกจุดที่เป็นไปได้ (ซึ่งพลาด
ได้ง่ายมากเมื่อโค้ดมีหลาย branch) แต่ RAII ทำให้เรื่องนี้กลายเป็น **หน้าที่ของ compiler**
โดยอัตโนมัติ: ไม่ว่าฟังก์ชันจะจบแบบไหน (return ปกติ หรือโยน exception ออกมากลางทาง) ตราบใดที่
object แบบ RAII ยังอยู่ใน scope ที่ยัง "สร้างสำเร็จแล้ว" destructor ของมัน **จะถูกเรียกเสมอ**

นี่คือเหตุผลที่แนวคิด RAII ถูกใช้เป็นรากฐานของโครงสร้างสำคัญเกือบทั้งหมดในไลบรารีมาตรฐานของ
C++ สมัยใหม่ — ตั้งแต่ `std::string`, `std::vector` (Module E) ไปจนถึง smart pointer อย่าง
`std::unique_ptr`/`std::shared_ptr` (Part 67) และ `std::lock_guard` สำหรับ mutex (Part 82)
ล้วนสร้างขึ้นจากหลักการเดียวกันกับ `SafeIntArray` ที่เราเพิ่งเขียนเองในหัวข้อนี้ทั้งสิ้น

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **คิดว่า compiler สร้าง default constructor ให้เสมอ** — ทันทีที่เขียน constructor แบบมี
   พารามิเตอร์เองสักตัวหนึ่ง compiler จะหยุดสร้าง default constructor ให้ ถ้ายังต้องการใช้
   `ClassName obj;` ต้องเขียน default constructor เพิ่มเข้าไปเอง (หรือใช้ `= default;`)
2. **assign ค่าใน body ของ constructor แทนที่จะใช้ member initializer list** — ทำงานถูกต้อง
   แต่เปลืองประสิทธิภาพโดยไม่จำเป็น (default-construct ก่อนแล้วค่อย assign ทับ) ควรใช้
   initializer list เป็นค่าเริ่มต้นเสมอสำหรับการกำหนดค่าตรงๆ
3. **เขียน initializer list โดยลำดับไม่ตรงกับลำดับการประกาศ member** — โดยเฉพาะเมื่อ member
   ตัวหนึ่งต้องพึ่งค่าของอีกตัวหนึ่ง จะเกิด Undefined Behavior เพราะ compiler initialize ตาม
   ลำดับการประกาศเสมอ ไม่ใช่ลำดับที่เขียนไว้ใน initializer list — สังเกต warning
   `-Wreorder` ที่ compiler แจ้งเตือนไว้แล้วเสมอ อย่าเพิกเฉย
4. **ปล่อยให้ compiler สร้าง copy constructor ให้ ทั้งที่ class มี raw pointer ที่เป็น
   เจ้าของหน่วยความจำเอง** — จะได้ Shallow Copy ที่ทำให้ object สองตัวชี้ไปยังหน่วยความจำ
   เดียวกัน นำไปสู่ **Double Free** ทันทีที่ทั้งสอง object ถูกทำลาย (หรือแย่กว่านั้นคือ
   **Dangling Pointer** ถ้า object ตัวหนึ่งถูกทำลายไปแล้วแต่อีกตัวยังใช้งาน pointer เดิมอยู่)
5. **ลืมว่า destructor รับพารามิเตอร์ไม่ได้และ overload ไม่ได้** — ต่างจาก constructor ที่
   overload ได้หลายแบบ class หนึ่งมี destructor ได้แค่ตัวเดียวเท่านั้น (`~ClassName()`)
6. **สร้าง object ด้วย `new` แล้วลืม `delete`** — destructor จะไม่ถูกเรียกเลยตราบใดที่
   โปรแกรมยังทำงานอยู่ ทำให้เกิด memory leak ทีละนิดสะสมไปเรื่อยๆ จนอาจทำให้โปรแกรมที่รันนาน
   (เช่น server) กินหน่วยความจำจนระบบล่มได้ในที่สุด
7. **เข้าใจผิดว่าต้องเขียน `delete` เองในทุกจุดที่ฟังก์ชันอาจ return** — ถ้าออกแบบ class ตาม
   หลัก RAII (ให้ destructor จัดการคืนทรัพยากรเอง) จะไม่ต้องกังวลเรื่องนี้เลย เพราะ destructor
   ถูกเรียกอัตโนมัติไม่ว่าฟังก์ชันจะจบแบบไหนก็ตาม

---

## แบบฝึกหัดท้ายบท

1. เขียน class `Employee` ที่มี member `std::string name` และ `double salary` (private
   ทั้งคู่) พร้อม constructor 3 แบบ (constructor overloading): ไม่รับ argument (ค่า default),
   รับแค่ชื่อ (เงินเดือนตั้งต้นที่ 15000), และรับทั้งชื่อกับเงินเดือน ทดสอบสร้าง object ทั้ง
   3 แบบใน `main`
2. เขียน class `Fraction` (เศษส่วน) ที่มี `const int numerator` และ `const int denominator`
   (ทั้งคู่กำหนดค่าได้ครั้งเดียวตอนสร้างเท่านั้น) พร้อม method `toDouble() const` ที่คำนวณค่า
   ทศนิยมของเศษส่วนนั้น อธิบายว่าทำไมต้องใช้ member initializer list กับ class นี้
3. ให้พิจารณาโค้ดต่อไปนี้ที่มีบั๊กเรื่องลำดับ initializer list แล้วแก้ไขให้ถูกต้อง:
   ```cpp
   class Rectangle {
       double area;
       double width;
       double height;
   public:
       Rectangle(double w, double h) : width(w), height(h), area(width * height) {}
   };
   ```
   (คำใบ้: ลองดูลำดับการประกาศ member เทียบกับลำดับใน initializer list ว่าตรงกันหรือไม่)
4. เขียน class `DynamicString` ที่ถือ `char*` เก็บข้อความ (จัดสรรด้วย `new[]` เอง) พร้อม
   เขียน **deep copy constructor** เองเพื่อป้องกันปัญหา double free ทดสอบด้วยการสร้าง object
   สองตัวจากการคัดลอกกัน แล้วพิมพ์ address ของ buffer ทั้งสองเพื่อยืนยันว่าต่างกันจริง
5. เขียน class `Tracer` (คล้ายตัวอย่างในหัวข้อ 46.7) ที่พิมพ์ข้อความตอนสร้างและตอนทำลาย
   object แล้วสร้าง object 4 ตัวในบล็อกเดียวกัน ทำนายผลลัพธ์ลำดับการพิมพ์ก่อนรันจริง แล้ว
   ตรวจสอบว่าตรงกับที่คาดไว้หรือไม่
6. เขียน class ตามแนวคิด RAII ชื่อ `ScopedResource` ที่พิมพ์ `"acquire resource"` ตอนสร้าง
   และพิมพ์ `"release resource"` ตอนทำลาย (ไม่ต้องจัดการหน่วยความจำจริงก็ได้ แค่พิมพ์ข้อความ
   พอ) แล้วสาธิตว่าแม้ฟังก์ชันจะ `return` กลางทางจากหลายจุด destructor ก็ยังถูกเรียกเสมอ

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <string>

class Employee {
private:
    std::string name;
    double salary;

public:
    Employee() : name("ไม่ระบุชื่อ"), salary(0.0) {
        std::cout << "เรียก Employee() default constructor\n";
    }

    Employee(const std::string& name) : name(name), salary(15000.0) {
        std::cout << "เรียก Employee(name) constructor\n";
    }

    Employee(const std::string& name, double salary) : name(name), salary(salary) {
        std::cout << "เรียก Employee(name, salary) constructor\n";
    }

    void printInfo() const {
        std::cout << "ชื่อ: " << name << ", เงินเดือน: " << salary << " บาท\n";
    }
};

int main() {
    Employee e1;
    Employee e2("สมหญิง");
    Employee e3("สมศักดิ์", 45000.0);

    e1.printInfo();
    e2.printInfo();
    e3.printInfo();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 exercise1.cpp -o exercise1
./exercise1
```

```
เรียก Employee() default constructor
เรียก Employee(name) constructor
เรียก Employee(name, salary) constructor
ชื่อ: ไม่ระบุชื่อ, เงินเดือน: 0 บาท
ชื่อ: สมหญิง, เงินเดือน: 15000 บาท
ชื่อ: สมศักดิ์, เงินเดือน: 45000 บาท
```

### แนวทางเฉลยข้อ 4

```cpp
#include <cstring>
#include <iostream>

class DynamicString {
private:
    char* buffer;

public:
    explicit DynamicString(const char* text) {
        std::size_t len = std::strlen(text);
        buffer = new char[len + 1];
        std::strcpy(buffer, text);
    }

    // Deep copy constructor: จัดสรร buffer ใหม่แล้วคัดลอกเนื้อหาไปทีละตัวอักษร
    DynamicString(const DynamicString& other) {
        std::size_t len = std::strlen(other.buffer);
        buffer = new char[len + 1];
        std::strcpy(buffer, other.buffer);
    }

    ~DynamicString() {
        delete[] buffer;
    }

    void print() const {
        std::cout << buffer << " (address: " << static_cast<const void*>(buffer) << ")\n";
    }
};

int main() {
    DynamicString a("สวัสดีชาว C++");
    DynamicString b = a; // เรียก deep copy constructor ที่เขียนเอง

    std::cout << "a: "; a.print();
    std::cout << "b: "; b.print();
    std::cout << "a และ b มี buffer คนละ address กัน ปลอดภัยจาก double free\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 exercise4.cpp -o exercise4
./exercise4
```

```
a: สวัสดีชาว C++ (address: 0x563fc71ce2b0)
b: สวัสดีชาว C++ (address: 0x563fc71ce2e0)
a และ b มี buffer คนละ address กัน ปลอดภัยจาก double free
```

ตรวจสอบด้วย `valgrind --leak-check=full` ยืนยันว่าไม่มี memory leak หรือ error ใดๆ เลย —
ทุก `new[]` มี `delete[]` คู่กันครบ และแต่ละ object เป็นเจ้าของหน่วยความจำของตัวเองอย่างแท้จริง

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจ **Default Constructor** และรู้ว่า compiler สร้างให้อัตโนมัติเมื่อไหร่ และหยุดสร้าง
  ให้เมื่อไหร่ (ทันทีที่มี parameterized constructor)
- เขียน **Parameterized Constructor** พร้อม **Constructor Overloading** เพื่อให้ class เดียว
  สร้าง object ได้หลายรูปแบบตามความต้องการ
- ใช้ **Member Initializer List** ได้อย่างถูกต้อง ทั้งในแง่ประสิทธิภาพ (หลีกเลี่ยงการ
  default-construct แล้ว assign ซ้ำ) และในแง่ความจำเป็น (สำหรับ `const`/reference member)
  พร้อมเข้าใจกฎสำคัญว่าต้อง initialize ตามลำดับการประกาศ ไม่ใช่ลำดับที่เขียนไว้
- แยกความแตกต่างระหว่าง **Shallow Copy** (default ของ compiler) กับ **Deep Copy** (เขียนเอง)
  และรู้ว่าเมื่อไหร่ต้องเขียน Copy Constructor เอง (เมื่อ class มี raw pointer ที่เป็นเจ้าของ
  หน่วยความจำเอง) เพื่อป้องกันปัญหา Double Free
- เข้าใจว่า **Destructor** ถูกเรียกเมื่อไหร่บ้าง (ออกจาก scope, `delete`, ลำดับ LIFO เมื่อมี
  หลาย object)
- เขียนโค้ดตามแนวคิด **RAII** ได้อย่างสมบูรณ์ ผ่านตัวอย่าง `SafeIntArray` ที่ผูกการจัดหาและ
  คืนทรัพยากรไว้กับ constructor/destructor โดยอัตโนมัติ

Constructor และ Destructor ที่เรียนใน Part นี้เป็นรากฐานสำคัญที่จะถูกใช้ซ้ำตลอดหลักสูตรที่
เหลือ โดยเฉพาะเมื่อเราเริ่มควบคุมการเข้าถึงข้อมูลอย่างเข้มงวดขึ้นด้วย **Access Specifier**
(`public`/`private`/`protected`) ใน Part ถัดไป และเมื่อไปถึง Move Semantics ใน Part 70 เรา
จะได้กลับมาขยายแนวคิด RAII และ Rule of Three ที่เกริ่นไว้ในหัวข้อ 46.6 ให้สมบูรณ์เป็น
**Rule of Five**

**ต่อไป:** [Part 47 — Encapsulation และ Access Specifier](./part-047-encapsulation.md)
