# Part 53: Static Member และ Class Design (Step 417–424)

> Module D — เริ่มต้น C++ และ OOP | Part 53 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 417–424
> Part ก่อนหน้า: [Part 52 — Friend Function และ Friend Class](./part-052-friend-functions.md) | Part ถัดไป: [Part 54 — Exception Handling ใน C++](./part-054-exceptions-cpp.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง **instance member** (ข้อมูลที่แต่ละ object มีสำเนาของตัวเอง) กับ
   **static member** (ข้อมูลที่ทุก object ของ class เดียวกันแชร์ร่วมกัน) ได้อย่างชัดเจน
2. ประกาศ static data member ภายใน class และ define มันนอก class ได้อย่างถูกต้อง เข้าใจว่าทำไม
   ต้อง define แยกต่างหาก (รวมถึงรู้จักทางลัดของ C++17 ด้วย `inline static`)
3. เขียนและเรียกใช้ static member function ทั้งผ่านชื่อ class โดยตรงและผ่าน object เข้าใจว่าทำไม
   static member function ไม่มี `this` pointer และเข้าถึง non-static member ไม่ได้
4. ออกแบบระบบนับจำนวน object ที่ยังมีชีวิตอยู่ (Object Counter) ด้วย static member ร่วมกับ
   constructor/destructor/copy constructor ได้ถูกต้องครบทุกกรณี
5. ใช้ `const static` และ `constexpr static` member เป็นค่าคงที่ระดับ compile-time เพื่อแทนที่
   Macro (`#define`) ที่เคยใช้ในภาษา C
6. ออกแบบ **Singleton Pattern** เบื้องต้นด้วย static member function และ static local variable
   (Meyer's Singleton) พร้อมเข้าใจว่าทำไมวิธีนี้ thread-safe ตั้งแต่ C++11
7. ตระหนักถึงข้อควรระวังเรื่อง static initialization order ระหว่างไฟล์ และผลกระทบต่อการออกแบบ
   class ที่ดี
8. สรุปหลักการออกแบบ Class ที่ดีที่เรียนมาตลอด Module D (Encapsulation, Inheritance,
   Polymorphism, Abstract Class, Operator Overloading, Friend, Static) และรู้ว่าเมื่อไหร่ควร
   และไม่ควรใช้ static member

---

## 53.1 static data member คืออะไร (Step 417)

ตลอด Module D ที่ผ่านมา ทุก member variable ที่เราประกาศใน class (เช่น `balance_` ใน
`BankAccount`, `title_` ใน `Book`) เป็น **instance member** — แปลว่าทุกครั้งที่สร้าง object ใหม่
ด้วย constructor จะมีการจอง memory สำหรับ member เหล่านี้แยกต่างหากสำหรับแต่ละ object

แต่บางครั้งเราต้องการข้อมูลที่ **"เป็นของ class เอง" ไม่ใช่ของ object ใดตัวหนึ่ง** เช่น:

- อัตราดอกเบี้ยที่ใช้ร่วมกันของบัญชีธนาคารทุกบัญชี (เปลี่ยนทีเดียว มีผลกับทุกบัญชี)
- จำนวน object ทั้งหมดที่เคยสร้างขึ้นมา (Object Counter)
- ค่าคงที่ที่เกี่ยวข้องกับ class เช่น ขนาดสูงสุดที่ยอมรับได้

นี่คือหน้าที่ของ **static data member** — ตัวแปรที่มีอยู่ **เพียงชุดเดียวในทั้งโปรแกรม**
ไม่ว่าจะสร้าง object ของ class นั้นกี่ตัวก็ตาม ทุก object จะ "มองเห็น" ค่าเดียวกัน

```
Object ปกติ (instance member)          static member (แชร์ร่วมกัน)
┌─────────────┐  ┌─────────────┐        ┌─────────────────────────┐
│ Account a   │  │ Account b   │        │ interestRate_ = 0.05    │
│ balance_=1000│  │ balance_=5000│  <───┤ (มีอยู่ชุดเดียว ไม่ว่าจะ │
└─────────────┘  └─────────────┘        │  มี object กี่ตัว)        │
      ▲                 ▲               └─────────────────────────┘
      └─────────────────┴──────── ทั้งสอง object เข้าถึง static member ตัวเดียวกัน
```

### กฎสำคัญ: ต้อง define static data member นอก class เสมอ

การประกาศ static member ใน class (`static double interestRate_;`) เป็นเพียง **การประกาศ
(declaration)** บอก compiler ว่า "มี member ชื่อนี้อยู่นะ" แต่ยังไม่ได้จองพื้นที่หน่วยความจำจริง
เราต้อง **define** มันอีกครั้งนอก class (โดยทั่วไปคือในไฟล์ `.cpp`) เพื่อให้ linker จองพื้นที่จริงให้

```cpp
#include <iostream>
#include <string>
#include <utility>

class BankAccount {
public:
    BankAccount(std::string owner, double balance)
        : owner_(std::move(owner)), balance_(balance) {}

    void addInterest() {
        balance_ += balance_ * interestRate_;
    }

    double getBalance() const { return balance_; }
    const std::string& getOwner() const { return owner_; }

    static void setInterestRate(double rate) { interestRate_ = rate; }
    static double getInterestRate() { return interestRate_; }

private:
    std::string owner_;
    double balance_;

    static double interestRate_;   // ประกาศ (declaration) เท่านั้น ยังไม่มีที่เก็บจริง
};

// ต้อง define นอก class เสมอ (ไม่งั้นจะได้ linker error: undefined reference)
double BankAccount::interestRate_ = 0.02;

int main() {
    BankAccount a("Somchai", 1000.0);
    BankAccount b("Malee", 5000.0);

    std::cout << "อัตราดอกเบี้ยเริ่มต้น: " << BankAccount::getInterestRate() << '\n';

    BankAccount::setInterestRate(0.05); // เปลี่ยนครั้งเดียว มีผลกับทุก object ที่มีอยู่และจะสร้างใหม่

    a.addInterest();
    b.addInterest();

    std::cout << a.getOwner() << ": " << a.getBalance() << '\n';
    std::cout << b.getOwner() << ": " << b.getBalance() << '\n';
    std::cout << "Interest rate (shared by all accounts): "
              << BankAccount::getInterestRate() << '\n';
}
```

คอมไพล์และรัน:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 bank_static.cpp -o bank_static
./bank_static
```

ผลลัพธ์:

```
อัตราดอกเบี้ยเริ่มต้น: 0.02
Somchai: 1050
Malee: 5250
Interest rate (shared by all accounts): 0.05
```

สังเกตว่าเราเรียก `BankAccount::setInterestRate(0.05)` **เพียงครั้งเดียว** แต่มีผลกับทั้ง `a`
และ `b` — เพราะ `interestRate_` ไม่ใช่ของ `a` หรือ `b` โดยเฉพาะ แต่เป็นของ class `BankAccount`
เอง

### ถ้าลืม define นอก class จะเกิดอะไรขึ้น

ลองลบบรรทัด `double BankAccount::interestRate_ = 0.02;` ออก แล้วคอมไพล์ดู (ตัวอย่างนี้จงใจ
สร้างบั๊กเพื่อสาธิต):

```cpp
#include <iostream>

class Counter {
public:
    static void increment() { count_++; }
    static int getCount() { return count_; }
private:
    static int count_;   // ประกาศไว้ แต่ "ลืม" define นอก class
};

int main() {
    Counter::increment();
    std::cout << Counter::getCount() << '\n';
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 counter_broken.cpp -o counter_broken
```

จะได้ **linker error** (ไม่ใช่ compile error) ประมาณนี้:

```
/usr/bin/ld: /tmp/xxx.o: in function `Counter::increment()':
counter_broken.cpp:(.text+0xa): undefined reference to `Counter::count_'
/usr/bin/ld: counter_broken.cpp:(.text+0x13): undefined reference to `Counter::count_'
collect2: error: ld returned 1 exit status
```

Error ประเภทนี้เกิดในขั้นตอน **Linking** (ทบทวนจาก Part 1) เพราะ compiler ยอมรับโค้ดว่าถูก
ไวยากรณ์ทุกอย่าง (มันแค่ "ประกาศว่ามี" `count_`) แต่ linker หาไม่เจอว่า `count_` ตัวจริงถูกจอง
memory ไว้ที่ไหน — นี่คือสัญญาณคลาสสิกของการลืม define static data member

---

## 53.2 static member function (Step 418)

เช่นเดียวกับ static data member, **static member function** คือฟังก์ชันที่ "เป็นของ class"
ไม่ใช่ของ object ใดตัวหนึ่ง จุดต่างสำคัญจาก member function ปกติ:

| คุณสมบัติ | Member Function ปกติ | Static Member Function |
|---|---|---|
| เรียกใช้ | ต้องผ่าน object (`obj.func()`) | เรียกผ่านชื่อ class ได้เลย (`Class::func()`) ไม่ต้องมี object |
| มี `this` pointer หรือไม่ | มี — ชี้ไปที่ object ที่เรียก | **ไม่มี** — เพราะไม่ผูกกับ object ใดเลย |
| เข้าถึง non-static member ได้หรือไม่ | ได้ (ผ่าน `this`) | **ไม่ได้** — ไม่รู้ว่าจะเข้าถึง object ไหน |
| เข้าถึง static member ได้หรือไม่ | ได้ | ได้ |
| ประกาศเป็น `const` ได้หรือไม่ | ได้ | **ไม่ได้** (ไม่มี object ให้ const อยู่แล้ว) |

จาก `BankAccount` ด้านบน `setInterestRate()` และ `getInterestRate()` เป็น static member
function ทั้งคู่ — สังเกตว่าเราเรียกมันผ่าน `BankAccount::setInterestRate(0.05)` โดยไม่ต้องมี
object ของ `BankAccount` เลยด้วยซ้ำก็ยังเรียกได้ (แม้ในตัวอย่างข้างบนเราจะมี `a` กับ `b` อยู่แล้ว
ก็ตาม)

### ทำไม static member function เข้าถึง instance member ไม่ได้

ลองดูตัวอย่างที่ตั้งใจให้ error (Step นี้สำคัญมากที่ต้องเข้าใจ "ทำไม" ไม่ใช่แค่ "จำกฎ"):

```cpp
#include <string>

class Robot {
public:
    Robot(std::string name) : name_(std::move(name)) { ++population_; }

    static int population() {
        return name_.size(); // ผิด: static function เข้าถึง instance member ไม่ได้
    }

private:
    std::string name_;
    static int population_;
};

int Robot::population_ = 0;
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -c robot_broken.cpp
```

Compiler จะฟ้อง error ชัดเจน:

```
robot_broken.cpp: In static member function 'static int Robot::population()':
robot_broken.cpp:8:16: error: invalid use of member 'Robot::name_' in static member function
    8 |         return name_.size(); // ผิด: static function เข้าถึง instance member ไม่ได้
      |                ^~~~~
robot_broken.cpp:12:17: note: declared here
   12 |     std::string name_;
      |                 ^~~~~
```

เหตุผลเชิงเทคนิค: member function ปกติทุกตัวจะได้รับ `this` pointer แบบซ่อนเป็นพารามิเตอร์ตัว
แรกเสมอ (นี่คือสิ่งที่ทำให้ `balance_ += ...` ใน `addInterest()` รู้ว่าเป็น `balance_` ของ object
ไหน) แต่ static member function **ไม่มี** `this` เพราะสามารถถูกเรียกได้โดยไม่มี object อยู่เลย
ด้วยซ้ำ (`Counter::increment()` ตอนที่ยังไม่เคยสร้าง `Counter` แม้แต่ตัวเดียวก็เรียกได้)
ดังนั้นภายในฟังก์ชันจึง**ไม่มีทางรู้ได้ว่าจะเข้าถึง `name_` ของ object ตัวไหน**

---

## 53.3 กรณีศึกษา: นับจำนวน Object ด้วย static (Object Counter) (Step 419)

หนึ่งในการใช้งาน static member ที่พบบ่อยที่สุดในโค้ดจริงคือการนับว่า **ขณะนี้มี object ของ
class นี้อยู่กี่ตัว** เทคนิคคือ:

1. เพิ่มค่าใน **constructor ทุกตัว** (default, parameterized, copy) เพราะทุก constructor คือ
   จุดที่ object ใหม่ถือกำเนิดขึ้น
2. ลดค่าใน **destructor** เพราะเป็นจุดที่ object สิ้นอายุ

```cpp
#include <iostream>
#include <string>
#include <utility>

class Widget {
public:
    explicit Widget(std::string label) : label_(std::move(label)) {
        ++liveCount_;
        std::cout << "  [สร้าง] " << label_ << " (มีอยู่ตอนนี้: " << liveCount_ << ")\n";
    }

    Widget(const Widget& other) : label_(other.label_ + "-copy") {
        ++liveCount_;
        std::cout << "  [copy]  " << label_ << " (มีอยู่ตอนนี้: " << liveCount_ << ")\n";
    }

    ~Widget() {
        --liveCount_;
        std::cout << "  [ทำลาย] " << label_ << " (เหลืออยู่: " << liveCount_ << ")\n";
    }

    static int liveCount() { return liveCount_; }

private:
    std::string label_;
    static int liveCount_;
};

int Widget::liveCount_ = 0;

void demoScope() {
    std::cout << "เข้า scope ย่อย\n";
    Widget w3("W3");
    Widget w4 = w3; // เรียก copy constructor
    std::cout << "จำนวน object ใน scope นี้: " << Widget::liveCount() << '\n';
    std::cout << "ออกจาก scope ย่อย\n";
}

int main() {
    std::cout << "จำนวน Widget ตอนเริ่มโปรแกรม: " << Widget::liveCount() << '\n';

    Widget w1("W1");
    Widget w2("W2");
    std::cout << "จำนวน Widget ตอนนี้: " << Widget::liveCount() << '\n';

    demoScope();

    std::cout << "จำนวน Widget หลังออกจาก scope ย่อย: " << Widget::liveCount() << '\n';
}
```

ผลลัพธ์:

```
จำนวน Widget ตอนเริ่มโปรแกรม: 0
  [สร้าง] W1 (มีอยู่ตอนนี้: 1)
  [สร้าง] W2 (มีอยู่ตอนนี้: 2)
จำนวน Widget ตอนนี้: 2
เข้า scope ย่อย
  [สร้าง] W3 (มีอยู่ตอนนี้: 3)
  [copy]  W3-copy (มีอยู่ตอนนี้: 4)
จำนวน object ใน scope นี้: 4
ออกจาก scope ย่อย
  [ทำลาย] W3-copy (เหลืออยู่: 3)
  [ทำลาย] W3 (เหลืออยู่: 2)
จำนวน Widget หลังออกจาก scope ย่อย: 2
  [ทำลาย] W2 (เหลืออยู่: 1)
  [ทำลาย] W1 (เหลืออยู่: 0)
```

จุดที่ต้องสังเกตให้ดี:

- `demoScope()` สร้าง `w3` แล้ว copy เป็น `w4` — ทำให้ `liveCount_` ขึ้นเป็น 4 ทั้งที่มีตัวแปร
  ชื่อ `w4` เพียงตัวเดียวที่มองเห็นในโค้ด (เพราะ `w4` แท้จริงมี object ของตัวเองที่ถูก copy มา)
- เมื่อออกจาก scope ของ `demoScope()` ทั้ง `w3` และ `w4` ถูกทำลายอัตโนมัติตามลำดับย้อนกลับ
  (LIFO — Last In, First Out เหมือน stack) ทำให้ `liveCount_` ลดกลับเหลือ 2 พอดี ตรงกับจำนวน
  `w1`, `w2` ที่ยังอยู่ใน `main`
- ถ้าเราลืมเพิ่ม copy constructor ที่เพิ่มค่า `liveCount_` (ปล่อยให้ compiler generate
  copy constructor แบบ default) ตัวนับจะ**ผิด** เพราะ default copy constructor จะ copy
  ค่า `label_` แต่ไม่รู้จักเพิ่ม `liveCount_` ให้ — นี่คือกับดักที่พบบ่อยมากในของจริง

---

## 53.4 const static และ constexpr static Member (Step 420)

อีกหนึ่งประโยชน์สำคัญของ static member คือการใช้แทน Macro (`#define`) ที่เราเคยใช้กำหนดค่าคงที่
ในภาษา C (ทบทวน Part 14) ข้อดีของการใช้ static const/constexpr member แทน macro คือ **มี type
ที่ชัดเจน อยู่ใน scope ของ class (ไม่ปนกับชื่ออื่นทั้งโปรแกรมแบบ macro) และ debugger มองเห็นได้**

```cpp
#include <iostream>
#include <string>
#include <type_traits>

class Circle {
public:
    explicit Circle(double radius) : radius_(radius) {}

    double area() const { return PI * radius_ * radius_; }

    // const static int/enum แบบดั้งเดิม: กำหนดค่าใน class ได้เลยถ้าเป็น integral type
    static const int MAX_RADIUS = 1000;

    // C++11: constexpr static ใช้ได้กับ integral/floating-point ที่เป็น literal type
    static constexpr double PI = 3.14159265358979;

private:
    double radius_;
};

class Library {
public:
    // C++17: inline static ทำให้ define ชนิดใดก็ได้ (รวม non-integral เช่น std::string)
    // ได้ในตัว class โดยไม่ต้อง define ซ้ำนอก class
    static inline const std::string SYSTEM_NAME = "CPP Course Library";
    static inline int totalBranches = 0;
};

int main() {
    Circle c(10.0);
    std::cout << "Area: " << c.area() << '\n';
    std::cout << "MAX_RADIUS (compile-time constant): " << Circle::MAX_RADIUS << '\n';

    static_assert(Circle::MAX_RADIUS == 1000, "ค่าคงที่ต้อง evaluate ได้ตอน compile");

    Library::totalBranches = 5;
    std::cout << Library::SYSTEM_NAME << " มี " << Library::totalBranches << " สาขา\n";
}
```

ผลลัพธ์:

```
Area: 314.159
MAX_RADIUS (compile-time constant): 1000
CPP Course Library มี 5 สาขา
```

### ตารางเปรียบเทียบวิธีประกาศ static constant ทั้งสามยุค

| วิธี | มาตรฐาน | ใช้กับชนิดใดได้ | ต้อง define นอก class หรือไม่ |
|---|---|---|---|
| `static const int X = N;` | C++98 | เฉพาะ integral/enum เท่านั้น | ไม่ต้อง ถ้าไม่มีการ "take address" ของมัน |
| `static constexpr T X = ...;` | C++11 | literal type (int, double, enum, ...) | ไม่ต้อง (ถือเป็น implicit inline) |
| `static inline T X = ...;` | C++17 | **ชนิดใดก็ได้** รวม `std::string`, `std::vector` ที่มี constexpr ctor หรือไม่ก็ได้ | ไม่ต้อง เพราะ `inline` การันตีว่านิยามซ้ำในหลายไฟล์ได้โดยไม่ error |

> ก่อน C++17 ถ้าต้องการ static member ที่เป็น `std::string` หรือชนิดอื่นที่ไม่ใช่ integral
> จำเป็นต้อง define นอก class เสมอ (เหมือน `interestRate_` ใน 53.1) คำสั่ง `inline` ใน C++17
> ทำให้ปัญหานี้หมดไป และเป็นวิธีที่**แนะนำที่สุด**สำหรับโค้ดใหม่ตั้งแต่ C++17 ขึ้นไป

---

## 53.5 Singleton Pattern เบื้องต้นด้วย static (Step 421)

**Singleton** คือ Design Pattern ที่ต้องการให้ class หนึ่ง **มี object ได้เพียงตัวเดียวเท่านั้น
ตลอดอายุโปรแกรม** และมีจุดเข้าถึง (access point) ที่เป็นสากล เช่น ระบบ Logger, ระบบตั้งค่า
(Configuration) หรือ Connection Pool ที่ควรมีอินสแตนซ์เดียวใช้ร่วมกันทั้งโปรแกรม

> Design Pattern ทั้งหมดอย่างละเอียด (Creational, Structural, Behavioral) จะเรียนเจาะลึกใน
> **Module H (Part 97-98)** ที่นี่เราจะเห็นแค่การประยุกต์ static member เพื่อทำ Singleton
> แบบง่ายที่สุดที่เรียกว่า **Meyer's Singleton** (ตั้งชื่อตาม Scott Meyers)

หลักการ: ใช้ **static local variable ภายใน static member function** — ตัวแปร static ภายใน
ฟังก์ชันจะถูกสร้างขึ้นเพียงครั้งเดียว (ตอนที่ฟังก์ชันถูกเรียกเป็นครั้งแรก) และคงอยู่ตลอดไปหลัง
จากนั้น

```cpp
#include <iostream>
#include <string>
#include <vector>

class Logger {
public:
    // ลบ copy constructor และ copy assignment เพื่อป้องกันการก็อปปี้ instance
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;

    static Logger& getInstance() {
        static Logger instance;   // สร้างครั้งแรกที่ถูกเรียกเท่านั้น (lazy init)
        return instance;          // C++11 การันตีว่า thread-safe
    }

    void log(const std::string& message) {
        history_.push_back(message);
        std::cout << "[LOG] " << message << '\n';
    }

    std::size_t entryCount() const { return history_.size(); }

private:
    Logger() = default;   // constructor เป็น private ห้ามสร้างจากภายนอกโดยตรง
    ~Logger() = default;

    std::vector<std::string> history_;
};

void doWork() {
    Logger::getInstance().log("doWork() เริ่มทำงาน");
}

int main() {
    Logger::getInstance().log("โปรแกรมเริ่มทำงาน");
    doWork();
    Logger::getInstance().log("โปรแกรมจบการทำงาน");

    std::cout << "จำนวน log ทั้งหมด: " << Logger::getInstance().entryCount() << '\n';
    std::cout << std::boolalpha
              << "getInstance() คืน object เดียวกันเสมอ: "
              << (&Logger::getInstance() == &Logger::getInstance()) << '\n';
}
```

ผลลัพธ์:

```
[LOG] โปรแกรมเริ่มทำงาน
[LOG] doWork() เริ่มทำงาน
[LOG] โปรแกรมจบการทำงาน
จำนวน log ทั้งหมด: 3
getInstance() คืน object เดียวกันเสมอ: true
```

### ส่วนประกอบสำคัญที่ทำให้เป็น Singleton ที่ถูกต้อง

1. **Constructor เป็น `private`**: ป้องกันไม่ให้ใครสร้าง `Logger` ด้วยตัวเองผ่าน `Logger l;`
   ได้โดยตรงจากภายนอก class
2. **ลบ copy constructor และ copy assignment** (`= delete`): ป้องกันการ copy instance ที่มีอยู่
   แล้วออกไปสร้างตัวสำเนาซ้ำ (ถ้าไม่ลบ จะ copy ได้และขัดกับหลักการ "มีตัวเดียว")
3. **static member function `getInstance()`**: เป็นจุดเข้าถึงเดียวที่อนุญาตให้สร้าง/เข้าถึง
   instance ได้ ภายในใช้ **static local variable** ซึ่งรับประกันว่าจะถูกสร้างครั้งเดียวเท่านั้น

---

## 53.6 Static Initialization Order และ Thread Safety (Step 422)

### ปัญหา "Static Initialization Order Fiasco"

ถ้า static data member ของ class ต่างกัน (หรือ global variable) อยู่คนละไฟล์ `.cpp` กัน
**มาตรฐาน C++ ไม่การันตีลำดับการ initialize ระหว่างไฟล์** (Translation Unit) — แปลว่าถ้า static
variable ในไฟล์ A ต้องใช้ค่าจาก static variable ในไฟล์ B ตอน initialize แต่ B ยังไม่ถูก
initialize ก่อน A จะเกิดปัญหาที่เรียกว่า **"Static Initialization Order Fiasco"**

ปัญหานี้เป็นสาเหตุคลาสสิกของบั๊กที่ debug ยากมาก เพราะบางครั้งโปรแกรมทำงานถูกในเครื่องหนึ่ง
(ลำดับ initialize บังเอิญถูก) แต่พังในอีกเครื่องหนึ่ง หรือพังหลัง compiler อัปเดตเวอร์ชัน

**วิธีแก้ที่นิยมที่สุด**: ใช้เทคนิคเดียวกับ Meyer's Singleton ใน 53.5 — เปลี่ยนจาก static
global/member variable ธรรมดา ให้เป็น **static local variable ภายในฟังก์ชัน** เพราะตัวแปรแบบนี้
จะถูก initialize ณ ครั้งแรกที่ฟังก์ชันถูกเรียกใช้งานจริง (lazy initialization) ไม่ใช่ตอนโปรแกรม
เริ่มทำงาน จึงไม่มีปัญหาเรื่องลำดับข้าม translation unit อีกต่อไป

### Thread Safety ของ static local variable

ตั้งแต่ **C++11** เป็นต้นไป มาตรฐานภาษาการันตีว่า **การ initialize static local variable เป็น
thread-safe โดยอัตโนมัติ** — ถ้ามีหลาย thread เรียก `Logger::getInstance()` พร้อมกันเป็นครั้ง
แรกในเวลาเดียวกัน compiler จะใส่กลไก locking ให้อัตโนมัติเพื่อรับประกันว่า `instance` จะถูก
สร้างเพียงครั้งเดียวเท่านั้น ไม่ว่าจะมีกี่ thread แข่งกันเรียกก็ตาม

นี่คือเหตุผลสำคัญที่ Meyer's Singleton เป็นวิธีการเขียน Singleton ที่**แนะนำที่สุด**ใน Modern
C++ (แทนที่วิธีเก่าที่ใช้ raw pointer + double-checked locking ที่เขียนถูกยากกว่ามาก) เรื่อง
Thread และ Concurrency แบบเจาะลึกจะเรียนใน **Module G (Part 81-85)**

---

## 53.7 หลักการออกแบบ Class ที่ดี: ทบทวน Module D ทั้งหมด (Step 423)

ก่อนจะจบ Module D เรามาทบทวนแนวคิดสำคัญของการออกแบบ class ที่เรียนมาทั้งหมด และดูว่า static
member เข้ากับภาพรวมตรงไหน:

| แนวคิด | Part ที่เรียน | สรุปสั้น |
|---|---|---|
| Class และ Object | 45 | โครงสร้างข้อมูล + พฤติกรรมรวมกันเป็นหน่วยเดียว |
| Constructor/Destructor | 46 | สร้าง/ทำลาย object อย่างปลอดภัย รากฐานของ RAII |
| Encapsulation | 47 | ซ่อนรายละเอียดภายใน เปิดเผยเฉพาะ interface ที่จำเป็น |
| Inheritance | 48 | สร้าง class ใหม่ต่อยอดจาก class เดิม (is-a relationship) |
| Polymorphism/Virtual | 49 | เรียกฟังก์ชันเดียวกัน แต่พฤติกรรมต่างกันตามชนิด object จริง |
| Abstract Class | 50 | กำหนด "สัญญา" (contract) ที่ class ลูกต้องทำตาม |
| Operator Overloading | 51 | ทำให้ object ใช้งานกับ operator ได้เป็นธรรมชาติเหมือน built-in type |
| Friend | 52 | อนุญาตพิเศษให้ฟังก์ชัน/class ภายนอกเข้าถึง private ได้ (ใช้อย่างประหยัด) |
| **Static Member** | **53 (Part นี้)** | ข้อมูล/พฤติกรรมที่เป็นของ class เอง ไม่ใช่ของ object |

หลักการออกแบบที่ดี (**Cohesion** และ **Single Responsibility**) บอกว่า class หนึ่งควรมีหน้าที่
รับผิดชอบเดียวที่ชัดเจน สมาชิกทุกตัว (ทั้ง instance และ static) ควรเกี่ยวข้องกับหน้าที่นั้น
โดยตรง ตัวอย่างเช่น `BankAccount::interestRate_` เหมาะสมเพราะเกี่ยวข้องกับ "การเป็นบัญชี
ธนาคาร" โดยตรง แต่ถ้าเราใส่ static member ที่ไม่เกี่ยวข้องกันเลย (เช่น สถิติการใช้ CPU ของทั้ง
โปรแกรม) เข้าไปใน `BankAccount` จะถือว่าออกแบบไม่ดี เพราะ `BankAccount` มีความรับผิดชอบเกิน
ขอบเขตของมัน

---

## 53.8 เมื่อไหร่ควรใช้ static Member และเมื่อไหร่ไม่ควร (Step 424)

### ควรใช้ static member เมื่อ:

1. **ข้อมูลนั้นเป็นคุณสมบัติร่วมของทุก object ในระดับ class จริงๆ** เช่น อัตราดอกเบี้ย, ค่าคงที่
   ทางฟิสิกส์, การตั้งค่าที่ใช้ร่วมกัน
2. **ต้องการนับหรือติดตามสถานะรวมของทุก object** เช่น Object Counter, resource pool ที่จำกัด
3. **สร้าง utility function ที่ไม่ต้องพึ่งข้อมูลของ object ใดตัวหนึ่ง** เช่น factory function,
   ฟังก์ชันแปลงหน่วย, `IdGenerator::nextId()`
4. **ต้องการ Singleton** ที่มี instance เดียวตลอดโปรแกรม

### ควรหลีกเลี่ยง หรือระวังเป็นพิเศษเมื่อ:

1. **static member ที่เป็น mutable state ที่ถูกแก้ไขบ่อย** — มันทำหน้าที่เหมือน **global
   variable ที่แอบซ่อนอยู่ใน class** ทำให้โค้ดคาดเดาพฤติกรรมยากขึ้น (side effect ที่มองไม่เห็น
   จากภายนอก) และทำให้ **เขียน unit test ยากขึ้นมาก** เพราะ test แต่ละเคสอาจกระทบ static state
   ที่ค้างจาก test เคสก่อนหน้า (Unit Testing แบบเจาะลึกจะเรียนใน Part 93)
2. **ใช้ static member แทนที่ควรจะเป็น parameter หรือ dependency injection** — ถ้าฟังก์ชันหนึ่ง
   ควรรับค่าจากภายนอกเป็น parameter แต่กลับไปดึงจาก static member แทน จะทำให้ทดสอบและนำโค้ด
   กลับมาใช้ซ้ำยากขึ้น
3. **Singleton ที่ถูกใช้พร่ำเพรื่อเกินไป** — Singleton ทำให้เกิด **global state โดยนัย** ซึ่งเป็น
   ที่ถกเถียงกันในวงการวิศวกรรมซอฟต์แวร์ว่าอาจนำไปสู่โค้ดที่ dependency ซ่อนอยู่มองไม่เห็น
   (จะพูดถึงข้อดีข้อเสียอย่างละเอียดใน Module H เรื่อง Design Pattern)

**กฎง่ายๆ ที่ใช้ได้เสมอ**: ถ้าลังเลว่าข้อมูลควรเป็น static หรือ instance member ให้ถามตัวเองว่า
"ข้อมูลนี้เปลี่ยนแปลงไปตาม object แต่ละตัวหรือไม่ ถ้าสร้าง object สองตัว ข้อมูลนี้ควรต่างกันไหม"
ถ้าคำตอบคือ "ควรต่างกัน" ให้ใช้ instance member ถ้าคำตอบคือ "ควรเหมือนกันเสมอไม่ว่าจะมีกี่
object" จึงค่อยพิจารณาใช้ static member

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ประกาศ static data member แต่ลืม define นอก class** — ได้ **linker error**
   (`undefined reference`) ไม่ใช่ compile error ทำให้บางคนงงว่าทำไม compile "ผ่าน" (จริงๆ
   compile ผ่าน แต่ link ไม่ผ่าน) วิธีแก้: ใช้ `static inline` (C++17) เพื่อไม่ต้องมาคอย define
   แยกอีกไฟล์เลย
2. **ลืมอัปเดต static counter ใน copy constructor** — ถ้า class มี custom copy constructor
   (เช่นใน 53.3) แต่ลืมเพิ่ม `++liveCount_` ในนั้น ตัวนับจะผิดพลาดทันทีที่มีการ copy object
   (นับน้อยกว่าจริง)
3. **คิดว่า static member function เข้าถึง instance member ได้เหมือน member function ปกติ** —
   จะได้ compile error `invalid use of member ... in static member function` ทันที เพราะไม่มี
   `this` pointer ให้อ้างอิง
4. **ใช้ static member เป็นตัวเก็บ state ที่ควรเป็นของแต่ละ object** — บั๊กที่พบบ่อยเมื่อสร้าง
   object หลายตัวแล้วพบว่าการเปลี่ยนค่าของตัวหนึ่งกระทบตัวอื่นโดยไม่ตั้งใจ ทั้งที่ควรจะเป็น
   ค่าเฉพาะของแต่ละ object
5. **สร้าง Singleton โดยใช้ raw pointer + `new` แบบเก่า** (`static Logger* instance; if
   (!instance) instance = new Logger();`) แทนที่จะใช้ static local variable — วิธีเก่านี้ไม่
   thread-safe โดยอัตโนมัติ (ต้องเขียน locking เอง) และยังทำให้เกิด memory leak เพราะไม่มีใคร
   เรียก `delete` เลยตลอดอายุโปรแกรม
6. **ใส่ static member มากเกินไปจนกลายเป็น "God Object"** — เมื่อ class หนึ่งเก็บ static state
   ของทั้งระบบไว้มากเกินไป จะทำให้ class นั้นกลายเป็นจุดศูนย์กลางที่ทุกส่วนของโปรแกรมต้องพึ่งพา
   (tight coupling) และยากต่อการทดสอบแยกส่วน

---

## แบบฝึกหัดท้ายบท

1. เขียน class `Employee` ที่มี static member นับจำนวนพนักงานทั้งหมดที่ยังมีชีวิตอยู่ในระบบ
   (เพิ่มใน constructor ลดใน destructor) พร้อม static member function `employeeCount()`
2. แก้ไข class `BankAccount` จาก 53.1 ให้เพิ่ม static member function `resetInterestRate()`
   ที่รีเซ็ตอัตราดอกเบี้ยกลับเป็นค่า default (0.02)
3. เขียน Singleton class ชื่อ `ConfigManager` ที่เก็บค่าตั้งค่าแบบ key-value ง่ายๆ (ใช้
   `std::map<std::string, std::string>`) ต้องไม่สามารถ copy instance ได้ และมี method
   `set(key, value)` กับ `get(key)`
4. อธิบายด้วยคำพูดตัวเองว่าทำไม static member function ไม่มี `this` pointer และเข้าถึง
   non-static member ไม่ได้ พร้อมยกตัวอย่างโค้ดสั้นๆ ที่ทำให้เกิด compile error จริง
5. ออกแบบ class `IdGenerator` ที่มี static member function ชื่อ `nextId()` ซึ่งคืนค่า id ที่ไม่
   ซ้ำกันทุกครั้งที่ถูกเรียก โดยใช้ static local variable ภายในฟังก์ชัน (ห้ามสร้าง object ของ
   `IdGenerator` ได้เลย — ลบ default constructor)
6. อภิปราย: static data member ต่างจาก global variable ธรรมดา (ที่ประกาศนอก class ทุก class)
   อย่างไร มีข้อดีอะไรเหนือกว่ากัน

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <string>
#include <utility>

class Employee {
public:
    Employee(std::string name, std::string position)
        : name_(std::move(name)), position_(std::move(position)) {
        ++employeeCount_;
    }

    ~Employee() {
        --employeeCount_;
    }

    const std::string& name() const noexcept { return name_; }
    const std::string& position() const noexcept { return position_; }

    static int employeeCount() noexcept { return employeeCount_; }

private:
    std::string name_;
    std::string position_;

    static int employeeCount_;
};

int Employee::employeeCount_ = 0;

int main() {
    std::cout << "จำนวนพนักงานตอนเริ่ม: " << Employee::employeeCount() << '\n';

    Employee e1("Somchai", "Developer");
    Employee e2("Malee", "Designer");
    {
        Employee e3("Kittipong", "Manager");
        std::cout << "จำนวนพนักงานใน scope ย่อย: " << Employee::employeeCount() << '\n';
    }
    std::cout << "จำนวนพนักงานหลังออกจาก scope ย่อย: " << Employee::employeeCount() << '\n';
}
```

ผลลัพธ์:

```
จำนวนพนักงานตอนเริ่ม: 0
จำนวนพนักงานใน scope ย่อย: 3
จำนวนพนักงานหลังออกจาก scope ย่อย: 2
```

จุดสำคัญ: เมื่อ `e3` หลุดออกจาก scope ย่อย destructor ถูกเรียกอัตโนมัติ ทำให้ `employeeCount_`
ลดกลับเหลือ 2 ตรงกับ `e1` และ `e2` ที่ยังอยู่ใน `main` — เป็นรูปแบบเดียวกับ Object Counter ใน
53.3 ทุกประการ เพียงแค่เปลี่ยนบริบทเป็นระบบพนักงาน

### แนวทางเฉลยข้อ 5

```cpp
#include <iostream>

class IdGenerator {
public:
    IdGenerator() = delete;   // ห้ามสร้าง object เพราะ class นี้เป็นแค่ utility (ทุกอย่างเป็น static)

    static int nextId() {
        static int counter = 1000;   // static local variable: คงค่าข้ามการเรียกแต่ละครั้ง สร้างครั้งเดียว
        return counter++;
    }
};

int main() {
    for (int i = 0; i < 5; ++i) {
        std::cout << "Generated ID: " << IdGenerator::nextId() << '\n';
    }
}
```

ผลลัพธ์:

```
Generated ID: 1000
Generated ID: 1001
Generated ID: 1002
Generated ID: 1003
Generated ID: 1004
```

จุดสำคัญ: `counter` เป็น **static local variable** (คนละแนวคิดกับ static data member ของ class
แต่ใช้กลไกเดียวกันคือ "สร้างครั้งเดียว คงอยู่ตลอดไป") มันถูก initialize เป็น `1000` เพียงครั้ง
เดียวตอนที่ `nextId()` ถูกเรียกเป็นครั้งแรก จากนั้นทุกครั้งที่เรียกซ้ำ ค่าที่เพิ่มไว้จากครั้งก่อน
จะยังอยู่ ไม่ถูกรีเซ็ตกลับเป็น 1000 ใหม่ — เทคนิคนี้เป็นรากฐานเดียวกับที่ Meyer's Singleton ใน
53.5 ใช้สร้าง instance เพียงครั้งเดียว การประกาศ `IdGenerator() = delete;` ยังสื่อความหมายให้
ผู้อ่านโค้ดเข้าใจทันทีว่า class นี้ถูกออกแบบมาให้ใช้เป็น utility ผ่าน static member เท่านั้น
ไม่มีเจตนาให้สร้าง object ใดๆ เลย

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้:

- **static data member** คือข้อมูลที่แชร์ร่วมกันของทุก object ใน class เดียวกัน ต้อง define
  นอก class เสมอ (ยกเว้นใช้ `inline static` ของ C++17)
- **static member function** ไม่มี `this` pointer เรียกได้โดยไม่ต้องมี object และเข้าถึงได้แค่
  static member เท่านั้น
- เทคนิค **Object Counter** ที่ใช้ static member ร่วมกับ constructor/destructor/copy
  constructor นับจำนวน object ที่ยังมีชีวิตอยู่ได้อย่างแม่นยำ
- **const static** และ **constexpr static** เป็นทางเลือกที่ดีกว่า macro สำหรับค่าคงที่ระดับ
  class โดย C++17 เพิ่ม `inline static` ให้ใช้กับชนิดใดก็ได้โดยไม่ต้อง define ซ้ำ
- **Meyer's Singleton** ใช้ static local variable สร้าง instance เดียวที่ thread-safe
  โดยอัตโนมัติตั้งแต่ C++11 — เกริ่นนำก่อนเจาะลึก Design Pattern ใน Module H
- ข้อควรระวังเรื่อง **static initialization order fiasco** และวิธีหลีกเลี่ยงด้วย lazy
  initialization
- ทบทวนภาพรวมการออกแบบ class ที่ดีตลอด Module D และหลักการตัดสินใจว่าเมื่อไหร่ควร/ไม่ควรใช้
  static member

ใน **Part 54** เราจะเปลี่ยนโฟกัสไปที่การจัดการข้อผิดพลาดแบบ Modern C++ อย่างเต็มรูปแบบผ่าน
**Exception Handling** — `try/catch/throw`, `std::exception` hierarchy, custom exception class,
และความสัมพันธ์ระหว่าง Exception กับ RAII ที่เราเรียนมาตั้งแต่ Part 46

**ต่อไป:** [Part 54 — Exception Handling ใน C++](./part-054-exceptions-cpp.md)
