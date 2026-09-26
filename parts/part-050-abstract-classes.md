# Part 50: Abstract Class และ Interface (Step 393–400)

> Module D — เริ่มต้น C++ และ OOP | Part 50 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 393–400
> Part ก่อนหน้า: [Part 49 — Polymorphism และ Virtual Function](./part-049-polymorphism-virtual.md) | Part ถัดไป: [Part 51 — Operator Overloading](./part-051-operator-overloading.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความหมายของ **Pure Virtual Function** (`= 0`) และเขียนมันได้อย่างถูกต้อง
2. อธิบายได้ว่า **Abstract Class** คืออะไร และทำไม C++ ถึงห้าม instantiate มันโดยตรง
3. ออกแบบ **Interface แบบ C++** (class ที่มีแต่ pure virtual function ล้วน) และเปรียบเทียบกับ
   `interface` ใน Java/C# ได้อย่างชัดเจน
4. สร้างระบบ Class Hierarchy ที่ใช้ Abstract Base Class จริง (`Shape` → `Circle`, `Rectangle`,
   `Triangle`) แล้วเรียกใช้งานผ่าน Polymorphism ได้อย่างถูกต้อง
5. เข้าใจว่า pure virtual function สามารถมี implementation ได้ และรู้ว่าเมื่อไหร่ควรใช้เทคนิคนี้
6. อธิบายและประยุกต์ใช้หลักการ **Dependency Inversion Principle (DIP)** เบื้องต้นในการออกแบบโค้ด
   ที่พึ่งพา abstraction แทนที่จะพึ่งพา concrete class โดยตรง

---

## 50.1 Pure Virtual Function คืออะไร (Step 393)

ใน Part 49 เราเรียนเรื่อง `virtual function` ไปแล้วว่ามันทำให้เกิด **Dynamic Dispatch** — เรียก
ฟังก์ชันผ่าน pointer/reference ของ base class แต่ได้พฤติกรรมของ derived class จริง คำถามที่ตามมา
คือ: ถ้า base class อย่าง `Shape` ไม่มีทางรู้เลยว่า "พื้นที่" ของรูปทรงทั่วไปคำนวณยังไง (เพราะ `Shape`
ไม่ใช่รูปทรงที่มีอยู่จริง เป็นแค่แนวคิดนามธรรม) เราจะเขียน `area()` ใน `Shape` ยังไงดี?

คำตอบคือ **Pure Virtual Function** — virtual function ที่ไม่มี implementation ใน base class
เลย และ "บังคับ" ให้ derived class ทุกตัวต้อง implement มันเอง ประกาศด้วยการเติม `= 0` ต่อท้าย
signature ของฟังก์ชัน:

```cpp
class Shape {
public:
    virtual double area() const = 0;   // pure virtual function
    virtual ~Shape() = default;
};
```

`= 0` ไม่ได้แปลว่า "คืนค่า 0" แต่เป็น **syntax พิเศษของภาษา** ที่บอกคอมไพเลอร์ว่า:

> "ฟังก์ชันนี้ไม่มี implementation ในคลาสนี้ ห้าม instantiate object ของคลาสนี้ตรงๆ
> เด็ดขาด และทุก class ที่สืบทอดไปต้อง override ฟังก์ชันนี้ ไม่งั้นจะกลายเป็น abstract
> class ต่อไปเรื่อยๆ"

Class ใดก็ตามที่มี pure virtual function อย่างน้อยหนึ่งตัว จะถูกเรียกว่า **Abstract Class**
โดยอัตโนมัติ — ไม่ต้องมี keyword พิเศษอะไรเพิ่มเติมเหมือนภาษาอื่น

```cpp
#include <iostream>

class Shape {
public:
    virtual double area() const = 0;   // pure virtual function
    virtual ~Shape() = default;
};

int main() {
    // Shape s;                 // ERROR: cannot declare variable 's' to be of abstract type 'Shape'
    std::cout << "Shape เป็น abstract class เพราะมี pure virtual function\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 shape_intro.cpp -o shape_intro
./shape_intro
# Shape เป็น abstract class เพราะมี pure virtual function
```

---

## 50.2 ทำไม Instantiate Abstract Class ไม่ได้ (Step 394)

ลองสั่ง `Shape s;` ดูจริงๆ (เอา comment ในตัวอย่างก่อนหน้าออก) จะได้ error ตอน compile ทันที
ไม่ใช่ warning และไม่ใช่ runtime error — คอมไพเลอร์ปฏิเสธไม่ให้โค้ดคอมไพล์ผ่านเลย:

```cpp
#include <iostream>

class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

int main() {
    Shape s;                    // พยายาม instantiate abstract class ตรงๆ
    std::cout << s.area() << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 shape_error.cpp -o shape_error
```

ผลลัพธ์:

```
shape_error.cpp: In function 'int main()':
shape_error.cpp:10:11: error: cannot declare variable 's' to be of abstract type 'Shape'
   10 |     Shape s;
      |           ^
shape_error.cpp:3:7: note:   because the following virtual functions are pure within 'Shape':
    3 | class Shape {
      |       ^~~~~
shape_error.cpp:5:20: note:     'virtual double Shape::area() const'
    5 |     virtual double area() const = 0;
      |                    ^~~~
```

**เหตุผลเชิงตรรกะ**: ถ้าคอมไพเลอร์ยอมให้สร้าง `Shape s;` ได้ แล้วมีใครมาเรียก `s.area()` จริงๆ
โปรแกรมจะต้องทำอะไร? ไม่มี implementation ของ `area()` อยู่ใน `Shape` เลยสักบรรทัดเดียว
มันจึงไม่มี Machine Code อะไรให้เรียกได้จริง คอมไพเลอร์จึงต้องปฏิเสธตั้งแต่ตอน compile time
เพื่อป้องกันไม่ให้เกิดสถานการณ์ที่เป็นไปไม่ได้แบบนี้

สิ่งที่ทำได้กับ abstract class มีอยู่ 2 อย่างเท่านั้น:

| ทำได้ | ทำไม่ได้ |
|---|---|
| ใช้เป็น **type ของ pointer/reference** เช่น `Shape*`, `Shape&` | สร้าง object ตรงๆ เช่น `Shape s;` |
| สืบทอด (`class Circle : public Shape`) แล้ว override ให้ครบทุก pure virtual function | สร้างผ่าน `new Shape()` |
| ใช้เป็น parameter type ของฟังก์ชัน (`void f(const Shape& s)`) | สร้างเป็น array `Shape arr[5];` |

ถ้า derived class **ไม่ได้ override** pure virtual function ให้ครบทุกตัว มันจะยังคงเป็น
abstract class ต่อไป และก็จะ instantiate ไม่ได้เหมือนกัน:

```cpp
class Shape {
public:
    virtual double area() const = 0;
    virtual double perimeter() const = 0;
};

class HalfDone : public Shape {
public:
    double area() const override { return 0.0; }
    // ลืม override perimeter() -> HalfDone ยังคงเป็น abstract class!
};

// HalfDone h;   // ยัง compile ไม่ผ่านเหมือนเดิม เพราะ perimeter() ยังไม่มี implementation
```

นี่คือกลไกที่คอมไพเลอร์ใช้ "บังคับสัญญา" (contract) ระหว่าง base class กับ derived class —
ถ้า derived class รับปากว่าจะเป็น "รูปทรงจริงๆ ที่สร้างได้" มันต้องทำตามสัญญาให้ครบทุกข้อ

---

## 50.3 การออกแบบ Interface แบบ C++ (Step 395)

C++ ไม่มี keyword `interface` เหมือน Java หรือ C# แต่เราสามารถจำลอง Interface ได้โดยการสร้าง
**abstract class ที่มีแต่ pure virtual function ล้วนๆ** และไม่มี data member ใดๆ เลย
(บาง guideline เรียก class แบบนี้ว่า "Pure Abstract Class" หรือ "Abstract Interface")

```cpp
class IPrintable {
public:
    virtual void print() const = 0;
    virtual ~IPrintable() = default;
};
```

ธรรมเนียมที่นิยมใช้กันในวงการ C++ (ไม่ใช่กฎบังคับของภาษา แต่เป็น convention ที่พบบ่อยมาก) คือ
ตั้งชื่อ interface ให้ขึ้นต้นด้วย `I` เช่น `IShape`, `ILogger`, `IPaymentMethod` เพื่อให้คนอ่านโค้ด
รู้ทันทีว่านี่คือ "สัญญา" ไม่ใช่ class ที่มี logic หรือ state จริงๆ

### กฎการออกแบบ Interface ที่ดี

1. **ห้ามมี data member** — interface เป็นแค่ "สัญญาว่าต้องทำอะไรได้บ้าง" ไม่ใช่ "เก็บข้อมูล
   อะไร" ถ้ามี data ที่ต้องแชร์ร่วมกันจริงๆ ให้ใช้ abstract class ธรรมดา (ไม่ต้องเป็น pure)
2. **ต้องมี virtual destructor เสมอ** — เพราะ object มักถูกลบผ่าน pointer ของ interface
   (`IShape* p = new Circle(...); delete p;`) ถ้าไม่มี virtual destructor destructor ของ
   `Circle` จะไม่ถูกเรียก ทำให้เกิด resource leak (รายละเอียดจะย้อนทวนใน 50.8)
3. **method ควรน้อยและเกี่ยวข้องกัน** — นี่คือหลักการ **Interface Segregation Principle (ISP)**
   หนึ่งใน SOLID Principles: อย่าออกแบบ interface ที่ใหญ่เทอะทะจนบังคับให้ทุก class ที่
   implement ต้องเขียน method ที่ตัวเองไม่ได้ใช้จริง
4. **ตั้งชื่อ method ด้วยคำกริยาที่สื่อถึง "พฤติกรรม"** ไม่ใช่ "โครงสร้างข้อมูล" เช่น `area()`,
   `send()`, `serialize()` ไม่ใช่ `data`, `value1`, `value2`

### เปรียบเทียบ Interface ใน C++ กับ Java/C#

| หัวข้อ | C++ (Pure Abstract Class) | Java `interface` | C# `interface` |
|---|---|---|---|
| Keyword เฉพาะ | ไม่มี ใช้ `class` ปกติ + pure virtual | มี `interface` | มี `interface` |
| Multiple Inheritance | ทำได้เต็มรูปแบบ (สืบทอดได้หลาย interface พร้อมกัน) | ทำได้ (`implements A, B`) | ทำได้ (`: IA, IB`) |
| Default Method | ทำได้ (pure virtual มี body ได้ ดู 50.6) | ทำได้ตั้งแต่ Java 8 (`default`) | ทำได้ตั้งแต่ C# 8 |
| Data Member | ทำได้ตามกฎภาษา แต่ **ไม่ควรมี** ตาม convention | ห้ามมี (มีแต่ `static final` constant) | ห้ามมี field ปกติ |
| ต้องประกาศชัดเจนว่า implement | ไม่ต้อง (ใช้ `: public IShape` เหมือน inheritance ปกติ) | ต้องใช้ `implements` | ต้องใช้ `:` |
| Virtual Destructor | ต้องเขียนเอง (สำคัญมาก!) | ไม่มีแนวคิด destructor (มี GC) | ไม่มีแนวคิด destructor (มี GC) |

ความแตกต่างที่สำคัญที่สุดคือ **C++ ไม่มี Garbage Collector** ดังนั้นเรื่อง virtual destructor
ใน interface ของ C++ จึงเป็นเรื่องที่ Java/C# programmer ที่เพิ่งย้ายมาเขียน C++ มักลืมและ
เจอ memory leak โดยไม่รู้ตัว

---

## 50.4 ตัวอย่างจริง: Shape Hierarchy — Base Class (Step 396)

มาสร้างระบบ Abstract Base Class ที่สมบูรณ์กัน โดยมี `Shape` เป็น abstract base class ที่กำหนด
"สัญญา" ว่ารูปทรงทุกชนิดต้องคำนวณ `area()`, `perimeter()` ได้ และมี "พฤติกรรมร่วม" อย่าง
`describe()` ที่ implement ไว้ใน base class เลย (ไม่ต้อง override ก็ได้ เพราะไม่ใช่ pure virtual)

```cpp
#include <iostream>
#include <string>

class Shape {
public:
    virtual double area() const = 0;
    virtual double perimeter() const = 0;

    // ฟังก์ชันนี้ "ไม่ใช่" pure virtual เพราะมี implementation สมบูรณ์อยู่แล้ว
    // มันเรียกใช้ area(), perimeter() และ name() ผ่าน dynamic dispatch (Template Method Pattern)
    virtual void describe() const {
        std::cout << "รูปทรงชนิด " << name() << " -> พื้นที่ = " << area()
                  << ", เส้นรอบรูป = " << perimeter() << '\n';
    }

    virtual ~Shape() = default;

protected:
    virtual std::string name() const = 0;   // pure virtual: derived class ต้อง implement เอง
};
```

สังเกตว่า `Shape` มี pure virtual function 3 ตัว (`area`, `perimeter`, `name`) แต่มี virtual
function ธรรมดาอีก 1 ตัว (`describe`) ที่ **ใช้งาน** pure virtual function เหล่านั้นภายในตัวมันเอง
นี่คือรูปแบบการออกแบบที่เรียกว่า **Template Method Pattern** — base class กำหนด "โครงของ
อัลกอริทึม" ไว้ ส่วน "รายละเอียดของแต่ละขั้นตอน" ให้ subclass เติมเข้ามาเอง เราจะเจอ pattern
นี้อย่างเป็นทางการอีกครั้งใน Part 97–98 (Design Pattern)

`name()` ถูกประกาศเป็น `protected` เพราะมันเป็นรายละเอียดภายในที่ใช้สำหรับพิมพ์ผลลัพธ์เท่านั้น
โค้ดภายนอกไม่ควรเรียก `shape.name()` ตรงๆ (ไม่ใช่ส่วนหนึ่งของ "public interface" ที่ผู้ใช้
ควรสนใจ) แต่ `describe()` ซึ่งเป็น member function ของ class เดียวกัน (แม้จะเรียกผ่าน `this`
ที่ชี้ไปยัง derived object จริง) มีสิทธิ์เรียก `protected` member ได้เสมอ

---

## 50.5 ตัวอย่างจริง: Circle, Rectangle, Triangle (Step 397)

ต่อจาก `Shape` เราสร้าง derived class 3 ตัวที่ implement pure virtual function ให้ครบทุกตัว:

```cpp
#include <cmath>

constexpr double kPi = 3.14159265358979323846;

class Circle : public Shape {
public:
    explicit Circle(double radius) : radius_(radius) {}
    double area() const override {
        return kPi * radius_ * radius_;
    }
    double perimeter() const override {
        return 2.0 * kPi * radius_;
    }

protected:
    std::string name() const override { return "Circle"; }

private:
    double radius_;
};

class Rectangle : public Shape {
public:
    Rectangle(double width, double height) : width_(width), height_(height) {}
    double area() const override { return width_ * height_; }
    double perimeter() const override { return 2.0 * (width_ + height_); }

protected:
    std::string name() const override { return "Rectangle"; }

private:
    double width_;
    double height_;
};

class Triangle : public Shape {
public:
    Triangle(double a, double b, double c) : a_(a), b_(b), c_(c) {}
    double area() const override {
        // สูตรของ Heron: s = ครึ่งหนึ่งของเส้นรอบรูป
        double s = perimeter() / 2.0;
        return std::sqrt(s * (s - a_) * (s - b_) * (s - c_));
    }
    double perimeter() const override { return a_ + b_ + c_; }

protected:
    std::string name() const override { return "Triangle"; }

private:
    double a_, b_, c_;
};
```

ทั้งสาม class implement `area()`, `perimeter()` และ `name()` ครบถ้วน (ทุกตัวเป็น `override`
ของ pure virtual function ใน `Shape`) จึงกลายเป็น **Concrete Class** ที่ instantiate ได้จริง
ต่างจาก `Shape` เอง

มาลองใช้งานผ่าน Polymorphism ด้วย `std::vector<std::unique_ptr<Shape>>` — เก็บ pointer ไปยัง
base class แต่แต่ละตัวชี้ไปยัง object ของ derived class คนละชนิดกัน:

```cpp
#include <memory>
#include <vector>

double total_area(const std::vector<std::unique_ptr<Shape>>& shapes) {
    double sum = 0.0;
    for (const auto& s : shapes) {
        sum += s->area();   // Dynamic Dispatch: เรียก area() ของ class จริงที่ s ชี้ไปหา
    }
    return sum;
}

int main() {
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>(3.0));
    shapes.push_back(std::make_unique<Rectangle>(4.0, 5.0));
    shapes.push_back(std::make_unique<Triangle>(3.0, 4.0, 5.0));

    for (const auto& s : shapes) {
        s->describe();
    }

    std::cout << "พื้นที่รวมทั้งหมด = " << total_area(shapes) << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 shapes.cpp -o shapes
./shapes
```

ผลลัพธ์:

```
รูปทรงชนิด Circle -> พื้นที่ = 28.2743, เส้นรอบรูป = 18.8496
รูปทรงชนิด Rectangle -> พื้นที่ = 20, เส้นรอบรูป = 18
รูปทรงชนิด Triangle -> พื้นที่ = 6, เส้นรอบรูป = 12
พื้นที่รวมทั้งหมด = 54.2743
```

สังเกตความสวยงามของโค้ดใน `main()` และ `total_area()`: **ไม่มีที่ไหนเลย** ที่ต้องเช็คว่า
"ถ้าเป็น Circle ให้ทำแบบนี้ ถ้าเป็น Rectangle ให้ทำแบบนั้น" (ไม่มี `if`/`switch` บน type) —
ทุกอย่างเกิดขึ้นอัตโนมัติผ่าน Virtual Table ที่เรียนใน Part 49 นี่คือพลังที่แท้จริงของ Abstract
Class ร่วมกับ Polymorphism: เราเขียนโค้ดที่ทำงานกับ "แนวคิดของ Shape" โดยไม่ต้องรู้จักรูปทรง
ที่จะถูกเพิ่มเข้ามาในอนาคตเลยด้วยซ้ำ ถ้าวันหนึ่งมีคนเพิ่ม `class Pentagon : public Shape`
เข้ามา โค้ดใน `total_area()` และ `main()` **ไม่ต้องแก้ไขแม้แต่บรรทัดเดียว**

---

## 50.6 Pure Virtual Function ที่มี Implementation ได้ (Step 398)

หลายคนเข้าใจผิดว่า pure virtual function (`= 0`) ต้อง "ว่างเปล่า" ไม่มี body เด็ดขาด แต่ความจริง
แล้ว **C++ อนุญาตให้ pure virtual function มี implementation ได้** โดยแยก declaration
(ที่มี `= 0`) กับ definition ออกจากกัน:

```cpp
#include <iostream>
#include <string>

class Logger {
public:
    // ยังคงเป็น pure virtual (= 0) จึงทำให้ Logger เป็น abstract class เหมือนเดิม
    // แต่ "มี" implementation เริ่มต้นให้ subclass เลือกเรียกใช้ผ่าน Logger::log() ได้
    virtual void log(const std::string& msg) const = 0;
    virtual ~Logger() = default;
};

void Logger::log(const std::string& msg) const {
    std::cout << "[LOG DEFAULT] " << msg << '\n';
}

class FileLogger : public Logger {
public:
    void log(const std::string& msg) const override {
        // เรียก implementation เริ่มต้นของ base class ได้ตรงๆ ด้วย Scope Resolution
        Logger::log(msg);
        std::cout << "[FileLogger] เขียนลงไฟล์เพิ่มเติมด้วย\n";
    }
};

int main() {
    FileLogger fl;
    fl.log("ระบบเริ่มทำงาน");
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 pure_body.cpp -o pure_body
./pure_body
```

ผลลัพธ์:

```
[LOG DEFAULT] ระบบเริ่มทำงาน
[FileLogger] เขียนลงไฟล์เพิ่มเติมด้วย
```

`Logger` ยังคงเป็น abstract class อยู่เหมือนเดิม (instantiate `Logger` ตรงๆ ไม่ได้) แต่ derived
class ที่ override `log()` แล้ว **ยังสามารถเรียกใช้ implementation เริ่มต้นของ base class ได้**
ผ่าน `Logger::log(msg)` เทคนิคนี้มีประโยชน์เมื่อต้องการบังคับให้ทุก subclass implement
ฟังก์ชันนี้เอง (เพราะเป็น `= 0`) แต่ก็อยากมี "โค้ดร่วม" ที่ subclass ส่วนใหญ่น่าจะอยากเรียกใช้
ด้วย ป้องกันการ copy-paste โค้ดซ้ำๆ กันในทุก subclass

> **ข้อควรระวัง**: เทคนิคนี้ใช้บ่อยไม่มากนักในโค้ดจริง เพราะทำให้โครงสร้างเข้าใจยากขึ้นเล็กน้อย
> (คนอ่านต้องรู้ว่า pure virtual มี body ได้) ถ้า logic ร่วมกันมีความสำคัญมาก ทางเลือกที่อ่าน
> ง่ายกว่าคือแยกเป็น non-virtual protected helper function อีกตัวต่างหาก เช่น
> `protected: void logToConsole(const std::string& msg) const;` แล้วให้ `log()` เรียกใช้
> จาก subclass เอง

---

## 50.7 Dependency Inversion Principle เบื้องต้น (Step 399)

**Dependency Inversion Principle (DIP)** เป็นตัว "D" ตัวสุดท้ายใน SOLID Principles (จะเรียน
ครบชุดใน Part 113–114 เรื่อง Software Architecture) หลักการนี้บอกว่า:

> "Module ระดับสูง (high-level) ไม่ควรพึ่งพา module ระดับล่าง (low-level) โดยตรง
> ทั้งสองฝ่ายควรพึ่งพา **abstraction** (interface) ร่วมกันแทน"

ฟังดูเป็นนามธรรม แต่ในทางปฏิบัติมันหมายถึงสิ่งที่เราเพิ่งเห็นใน `total_area()` ไปแล้ว: ฟังก์ชันนี้
"พึ่งพา" (depend on) แค่ `Shape` (abstraction) ไม่ได้พึ่งพา `Circle` หรือ `Rectangle`
(concrete class) โดยตรงเลย

มาดูตัวอย่างที่ชัดเจนกว่านั้นในบริบทของระบบแจ้งเตือน สมมติเรามีระบบสั่งซื้อสินค้าที่ต้องส่ง
การแจ้งเตือนหลังสั่งซื้อสำเร็จ:

```cpp
#include <iostream>
#include <memory>
#include <string>

// Interface (abstract class ที่มีแต่ pure virtual function ล้วน)
class INotifier {
public:
    virtual void send(const std::string& message) const = 0;
    virtual ~INotifier() = default;
};

class EmailNotifier : public INotifier {
public:
    void send(const std::string& message) const override {
        std::cout << "[Email] ส่งข้อความ: " << message << '\n';
    }
};

class SmsNotifier : public INotifier {
public:
    void send(const std::string& message) const override {
        std::cout << "[SMS] ส่งข้อความ: " << message << '\n';
    }
};

// OrderService ไม่รู้จัก EmailNotifier หรือ SmsNotifier เลย
// รู้จักแค่ "อะไรก็ได้ที่เป็น INotifier" -> นี่คือ Dependency Inversion Principle
class OrderService {
public:
    explicit OrderService(std::unique_ptr<INotifier> notifier)
        : notifier_(std::move(notifier)) {}

    void place_order(const std::string& item) const {
        std::cout << "สั่งซื้อสินค้า: " << item << '\n';
        notifier_->send("คำสั่งซื้อ '" + item + "' ได้รับการยืนยันแล้ว");
    }

private:
    std::unique_ptr<INotifier> notifier_;
};

int main() {
    OrderService order_by_email(std::make_unique<EmailNotifier>());
    order_by_email.place_order("คีย์บอร์ดกลไก");

    OrderService order_by_sms(std::make_unique<SmsNotifier>());
    order_by_sms.place_order("เมาส์ไร้สาย");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 dip_demo.cpp -o dip_demo
./dip_demo
```

ผลลัพธ์:

```
สั่งซื้อสินค้า: คีย์บอร์ดกลไก
[Email] ส่งข้อความ: คำสั่งซื้อ 'คีย์บอร์ดกลไก' ได้รับการยืนยันแล้ว
สั่งซื้อสินค้า: เมาส์ไร้สาย
[SMS] ส่งข้อความ: คำสั่งซื้อ 'เมาส์ไร้สาย' ได้รับการยืนยันแล้ว
```

### ทำไมการออกแบบแบบนี้ถึงดีกว่า?

ลองเปรียบเทียบกับแบบที่ **ไม่** ใช้ DIP: ถ้า `OrderService` เขียนขึ้นมาให้ผูกติดกับ
`EmailNotifier` โดยตรง (เช่นมี member เป็น `EmailNotifier notifier_;` ตรงๆ) จะเกิดปัญหา:

| ปัญหาแบบไม่มี DIP | ผลลัพธ์เมื่อใช้ DIP (พึ่งพา `INotifier`) |
|---|---|
| อยากเปลี่ยนไปแจ้งเตือนผ่าน SMS ต้องแก้โค้ดใน `OrderService` เอง | สร้าง `SmsNotifier` ใหม่ แล้วส่งเข้ามาตอนสร้าง `OrderService` เลย ไม่ต้องแก้ `OrderService` |
| เขียน Unit Test ยาก เพราะทุกครั้งที่ test ต้องส่ง Email จริง | สร้าง `MockNotifier` ปลอมขึ้นมา implement `INotifier` แล้วส่งเข้าไปแทนได้ทันที |
| `OrderService` "รู้" รายละเอียดภายในของการส่ง Email มากเกินไป | `OrderService` รู้แค่ "ส่งข้อความได้" ไม่สนใจว่าข้างในทำงานยังไง |

การพึ่งพา abstraction แทน concrete class ทำให้โค้ดเปิดกว้างสำหรับการขยาย (เพิ่ม
`PushNotifier`, `LineNotifier` ในอนาคต) แต่ปิดกั้นไม่ให้ต้องแก้โค้ดเดิมที่ทำงานอยู่แล้ว — นี่
คือหลักการ **Open/Closed Principle** (ตัว "O" ใน SOLID) ที่ทำงานร่วมกับ DIP อย่างแนบแน่น
เราจะกลับมาเจาะลึกเรื่อง SOLID ทั้งหมดอย่างเป็นทางการใน Part 113–114

---

## 50.8 Virtual Destructor ใน Abstract Class: ทบทวนให้ลึกขึ้น (Step 400)

Part 49 เคยพูดถึง virtual destructor ไปบ้างแล้ว แต่สำหรับ Abstract Class/Interface เรื่องนี้
สำคัญมากจนต้องย้ำอีกครั้งด้วยตัวอย่างที่ชัดเจนที่สุด: **ทุก abstract class ที่จะถูกใช้งานผ่าน
pointer ต้องมี virtual destructor เสมอ ไม่มีข้อยกเว้น**

```cpp
#include <iostream>

// เวอร์ชันที่ "ผิด": destructor ไม่ใช่ virtual
class BadBase {
public:
    virtual void work() const = 0;
    ~BadBase() { std::cout << "BadBase destructor\n"; }   // ไม่มี virtual!
};

class BadDerived : public BadBase {
public:
    void work() const override {}
    ~BadDerived() { std::cout << "BadDerived destructor (จัดการทรัพยากรบางอย่าง)\n"; }
};

int main() {
    BadBase* p = new BadDerived();
    delete p;   // Undefined Behavior: destructor ของ BadDerived อาจไม่ถูกเรียก!
    return 0;
}
```

ในตัวอย่างนี้ เมื่อเรียก `delete p` ผ่าน pointer ชนิด `BadBase*` แต่ `~BadBase()` ไม่ใช่
`virtual` คอมไพเลอร์จะ **bind การเรียก destructor แบบ static** (ตัดสินใจตอน compile time
จาก type ของ pointer คือ `BadBase*`) ไม่ใช่ dynamic dispatch เหมือน virtual function ทั่วไป
ผลคือ `~BadDerived()` อาจไม่ถูกเรียกเลย — ถ้า `BadDerived` มีการจอง memory หรือทรัพยากรอื่นๆ
ไว้ใน constructor สิ่งเหล่านั้นจะไม่ถูกคืนกลับ กลายเป็น **Resource Leak** ที่ตรวจจับยากมาก
เพราะโปรแกรมยังคง compile และ run ได้ตามปกติ ไม่มี error ใดๆ ให้เห็นทันที

วิธีแก้คือประกาศ destructor เป็น `virtual` เสมอ (แบบที่เราทำมาตลอดใน `Shape`, `INotifier`,
`Logger`):

```cpp
class GoodBase {
public:
    virtual void work() const = 0;
    virtual ~GoodBase() = default;   // virtual destructor: ปลอดภัย
};
```

**กฎทองที่ต้องจำ**: ถ้า class ของคุณมี virtual function อย่างน้อยหนึ่งตัว (ซึ่งรวมถึงทุก
abstract class อยู่แล้วโดยธรรมชาติ) **ให้ประกาศ destructor เป็น `virtual` เสมอ** แม้ว่า
destructor นั้นจะไม่มีอะไรให้ทำเลยก็ตาม (`= default` ก็เพียงพอ)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมใส่ `virtual` หน้า destructor ของ abstract class** — ตามที่อธิบายใน 50.8 นี่คือ
   ข้อผิดพลาดที่อันตรายที่สุดในหัวข้อนี้ เพราะไม่มี compiler error หรือ warning ใดๆ เตือน
   (บาง compiler อย่าง GCC จะเตือนด้วย `-Wnon-virtual-dtor` ถ้ามี `-Wextra` แต่ต้องสังเกตดีๆ)
2. **เข้าใจผิดว่า pure virtual function ต้องไม่มี implementation** — ตามที่เห็นใน 50.6
   จริงๆ แล้วมันมี body ได้ เพียงแต่ derived class ยังคงต้อง override เสมอ
3. **ลืม override pure virtual function ให้ครบทุกตัว** — ทำให้ derived class กลายเป็น
   abstract class โดยไม่ตั้งใจ แล้วงงว่าทำไม instantiate ไม่ได้ ให้อ่าน error message ของ
   compiler ให้ละเอียด มันจะบอกชื่อฟังก์ชันที่ยังเป็น pure อยู่เสมอ
4. **ออกแบบ Interface ใหญ่เกินไป (Fat Interface)** — เช่นสร้าง `IShape` ที่มีทั้ง
   `area()`, `perimeter()`, `draw()`, `serialize()`, `to_json()` รวมกันหมด ทำให้ class ที่
   implement ต้องเขียน method ที่ตัวเองไม่เกี่ยวข้องเลย (ขัดกับ Interface Segregation
   Principle) ควรแยกเป็นหลาย interface เล็กๆ ตามความรับผิดชอบแทน
5. **ใส่ data member เข้าไปใน "interface"** — ทำให้เส้นแบ่งระหว่าง "สัญญา" กับ
   "implementation" เบลอ ควรเก็บ data ไว้ที่ concrete class เท่านั้น ถ้าจำเป็นต้องมี data
   ร่วมกันจริงๆ ให้ใช้ abstract class ธรรมดา (ไม่ต้อง pure ทั้งหมด) แทน
6. **สับสนระหว่าง Abstract Class กับ Interface** — ใน C++ ทั้งสองคำนี้ไม่มีเส้นแบ่งทางภาษา
   ที่ชัดเจน (ต่างจาก Java) "Interface" เป็นแค่ชื่อเรียกที่ใช้กับ abstract class ที่บังเอิญ
   ไม่มี data member และมีแต่ pure virtual function ล้วน ในขณะที่ "Abstract Class" ทั่วไป
   อาจมีทั้ง data member, non-virtual function, และ virtual function ที่ไม่ใช่ pure ปนกันได้

---

## แบบฝึกหัดท้ายบท

1. เพิ่ม `class Square` เข้าไปใน hierarchy ของ `Shape` (จะสืบทอดจาก `Shape` โดยตรง หรือจาก
   `Rectangle` ก็ได้ ลองคิดว่าแบบไหนสมเหตุสมผลกว่ากันและเพราะอะไร)
2. เขียน abstract class `Animal` ที่มี pure virtual function `make_sound()` แล้วสร้าง
   `class Dog` กับ `class Cat` ที่สืบทอดมา ทดสอบเรียกผ่าน `std::vector<std::unique_ptr<Animal>>`
3. อธิบายด้วยคำพูดตัวเอง (ไม่ต้องเขียนโค้ด) ว่าทำไม copy constructor ของ abstract class ถึง
   "ไม่มีปัญหา" ทั้งที่เราสร้าง object ของ abstract class ตรงๆ ไม่ได้ (คำใบ้: ใครเป็นคนเรียก
   copy constructor ของ base class จริงๆ?)
4. ลองลบ `virtual` ออกจาก destructor ของ `Shape` ในตัวอย่างข้อ 50.5 แล้วเขียนโค้ดทดสอบที่
   สร้าง `Circle` ผ่าน `Shape*` แล้ว `delete` ทิ้ง (ใส่ `std::cout` ใน destructor ของ `Circle`
   ด้วยเพื่อดูว่ามันถูกเรียกหรือไม่)
5. ออกแบบ interface `IPaymentMethod` ที่มี pure virtual function `pay(double amount)` และ
   `name()` แล้วสร้าง `CreditCard` กับ `PayPal` implement มัน จากนั้นเขียน class `Checkout`
   ที่รับ `const IPaymentMethod&` เป็น parameter (สาธิตหลักการ Dependency Inversion)
6. อธิบายว่าทำไม method `name()` ใน `Shape` (ตัวอย่าง 50.4) ถึงถูกประกาศเป็น `protected`
   แทนที่จะเป็น `public` และมันจะเกิดปัญหาอะไรถ้าเปลี่ยนเป็น `private` แทน

### แนวทางเฉลยข้อ 2

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <string>

class Animal {
public:
    explicit Animal(std::string name) : name_(std::move(name)) {}

    virtual void make_sound() const = 0;

    void introduce() const {
        std::cout << name_ << " ร้องว่า: ";
        make_sound();
    }

    virtual ~Animal() = default;

protected:
    std::string name_;
};

class Dog : public Animal {
public:
    explicit Dog(std::string name) : Animal(std::move(name)) {}

    void make_sound() const override {
        std::cout << "โฮ่ง โฮ่ง!\n";
    }
};

class Cat : public Animal {
public:
    explicit Cat(std::string name) : Animal(std::move(name)) {}

    void make_sound() const override {
        std::cout << "เมี้ยว~\n";
    }
};

int main() {
    std::vector<std::unique_ptr<Animal>> animals;
    animals.push_back(std::make_unique<Dog>("โบ๊ท"));
    animals.push_back(std::make_unique<Cat>("มะลิ"));

    for (const auto& a : animals) {
        a->introduce();
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 animal.cpp -o animal
./animal
```

ผลลัพธ์:

```
โบ๊ท ร้องว่า: โฮ่ง โฮ่ง!
มะลิ ร้องว่า: เมี้ยว~
```

`Animal` เป็น abstract class เพราะมี `make_sound()` เป็น pure virtual แต่ `introduce()`
เป็น virtual function ธรรมดาที่ implement ไว้แล้วใน base class (คล้ายกับ `describe()` ใน
`Shape`) และใช้ `make_sound()` ที่เป็น pure virtual ภายในตัวมันเอง — เป็น Template Method
Pattern อีกครั้งหนึ่ง

### แนวทางเฉลยข้อ 5

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <string>

class IPaymentMethod {
public:
    virtual bool pay(double amount) const = 0;
    virtual std::string name() const = 0;
    virtual ~IPaymentMethod() = default;
};

class CreditCard : public IPaymentMethod {
public:
    explicit CreditCard(std::string number) : number_(std::move(number)) {}

    bool pay(double amount) const override {
        std::cout << "ตัดเงินผ่านบัตรเครดิตเลขท้าย "
                  << number_.substr(number_.size() - 4) << " จำนวน " << amount << " บาท\n";
        return true;
    }

    std::string name() const override { return "Credit Card"; }

private:
    std::string number_;
};

class PayPal : public IPaymentMethod {
public:
    explicit PayPal(std::string email) : email_(std::move(email)) {}

    bool pay(double amount) const override {
        std::cout << "ตัดเงินผ่าน PayPal (" << email_ << ") จำนวน " << amount << " บาท\n";
        return true;
    }

    std::string name() const override { return "PayPal"; }

private:
    std::string email_;
};

// Checkout ไม่รู้จัก CreditCard หรือ PayPal เลย รู้จักแค่ IPaymentMethod
class Checkout {
public:
    static bool process(const IPaymentMethod& method, double amount) {
        std::cout << "กำลังชำระเงินผ่าน " << method.name() << "...\n";
        return method.pay(amount);
    }
};

int main() {
    CreditCard card("4111111111111234");
    PayPal paypal("user@example.com");

    Checkout::process(card, 590.0);
    Checkout::process(paypal, 250.0);

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 payment.cpp -o payment
./payment
```

ผลลัพธ์:

```
กำลังชำระเงินผ่าน Credit Card...
ตัดเงินผ่านบัตรเครดิตเลขท้าย 1234 จำนวน 590 บาท
กำลังชำระเงินผ่าน PayPal...
ตัดเงินผ่าน PayPal (user@example.com) จำนวน 250 บาท
```

`Checkout::process` รับ `const IPaymentMethod&` เป็น parameter — มันไม่จำเป็นต้องรู้จัก
`CreditCard` หรือ `PayPal` เลยแม้แต่น้อย ถ้าวันหนึ่งมีคนเพิ่ม `class BankTransfer :
public IPaymentMethod` เข้ามา `Checkout` ก็ยังทำงานได้ทันทีโดยไม่ต้องแก้ไขโค้ดเลยสักบรรทัด
นี่คือ Dependency Inversion Principle ที่ทำงานจริงในระบบที่ใกล้เคียงกับงานจริงมากขึ้น

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจความหมายของ Pure Virtual Function (`= 0`) และรู้ว่ามันบังคับให้ derived class ต้อง
  implement เอง ไม่งั้นจะกลายเป็น abstract class ต่อไปเรื่อยๆ
- เข้าใจเหตุผลเชิงลึกว่าทำไม C++ ห้าม instantiate abstract class โดยตรง (ไม่มี Machine Code
  ให้เรียกจริง)
- ออกแบบ Interface แบบ C++ ได้ (abstract class ที่มีแต่ pure virtual function ล้วน) และรู้จัก
  ความแตกต่างจาก `interface` ของ Java/C# อย่างชัดเจน
- สร้างระบบ `Shape` → `Circle`/`Rectangle`/`Triangle` ที่ใช้งานผ่าน Polymorphism ได้จริง
  พร้อมเข้าใจ Template Method Pattern เบื้องต้น
- รู้ว่า pure virtual function มี implementation ได้ และรู้ขอบเขตว่าเมื่อไหร่ควรใช้เทคนิคนี้
- เข้าใจและประยุกต์ใช้ Dependency Inversion Principle เบื้องต้น: การพึ่งพา abstraction
  แทนที่จะพึ่งพา concrete class โดยตรง ทำให้โค้ดขยายง่ายและทดสอบง่ายขึ้นมาก
- ย้ำความสำคัญของ virtual destructor ใน abstract class อีกครั้งด้วยตัวอย่างที่แสดง
  Undefined Behavior จริงเมื่อลืมใส่

ใน **Part 51** เราจะเรียนเรื่อง **Operator Overloading** — การสอนให้ operator มาตรฐาน
อย่าง `+`, `-`, `==`, `<<`, `[]`, `++` ทำงานกับ class ของเราเองได้อย่างเป็นธรรมชาติ
พร้อมเจาะลึกว่าเมื่อไหร่ควรเขียนเป็น member function และเมื่อไหร่ต้องเขียนเป็น non-member
(friend) function

**ต่อไป:** [Part 51 — Operator Overloading](./part-051-operator-overloading.md)
