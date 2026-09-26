# Part 54: Exception Handling ใน C++ (Step 425–432)

> Module D — เริ่มต้น C++ และ OOP | Part 54 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 425–432
> Part ก่อนหน้า: [Part 53 — Static Member และ Class Design](./part-053-static-members.md) | Part ถัดไป: [Part 55 — โปรเจกต์ Library Management System](./part-055-library-system-project.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายกลไก `try`/`catch`/`throw` ของ C++ ได้อย่างละเอียด และเข้าใจว่ามันแตกต่างจากวิธีจัดการ
   error แบบ return code ที่เคยใช้ใน C (ทบทวน Part 16) อย่างไร
2. `throw` object ที่มีข้อมูลประกอบ (ไม่ใช่แค่ primitive type อย่างตัวเลขหรือ string ธรรมดา)
   เพื่อสื่อสารบริบทของข้อผิดพลาดได้ครบถ้วน
3. เข้าใจโครงสร้าง **std::exception hierarchy** ของ Standard Library และรู้จัก exception
   มาตรฐานที่ใช้บ่อย เช่น `std::runtime_error`, `std::logic_error`, `std::out_of_range`,
   `std::invalid_argument`
4. เขียน **custom exception class** ของตัวเองที่สืบทอดจาก `std::exception` (หรือ class ลูกของ
   มัน) พร้อม override `what()` ได้อย่างถูกต้องและปลอดภัย
5. เขียน `catch` หลายชนิดเรียงลำดับถูกต้อง (เฉพาะเจาะจงมาก่อนทั่วไป) และใช้ `catch(...)` เพื่อ
   ดักจับทุกสิ่งที่เหลือได้อย่างเหมาะสม
6. อธิบายว่าทำไม **RAII** (ที่เรียนใน Part 46) เป็นกลไกสำคัญที่สุดที่ทำให้ C++ จัดการ resource
   ได้ปลอดภัยแม้เกิด exception ผ่านกระบวนการ **Stack Unwinding**
7. ใช้ `noexcept` keyword ระบุว่าฟังก์ชันจะไม่ throw exception และเข้าใจผลกระทบต่อ
   performance และพฤติกรรมของ Standard Library (เช่น `std::vector`)
8. ตระหนักถึงข้อควรระวังสำคัญที่สุดข้อหนึ่งของ C++: **ห้าม throw exception ออกจาก destructor**
   และรู้ว่าจะเกิดอะไรขึ้นถ้าฝ่าฝืน

---

## 54.1 try/catch/throw พื้นฐาน (Step 425)

ใน Part 16 เราเรียนวิธีจัดการ error แบบภาษา C คือการ**คืนค่า error code** จากฟังก์ชัน (เช่น
`return -1;`) แล้วให้ผู้เรียกตรวจสอบค่านั้นเอง วิธีนี้มีจุดอ่อนสำคัญคือ **ผู้เขียนโค้ดสามารถลืม
ตรวจสอบ error code ได้ง่ายมาก** และโปรแกรมจะทำงานต่อไปทั้งที่มีข้อผิดพลาดเกิดขึ้นแล้ว

C++ มีกลไกที่ทรงพลังกว่าเรียกว่า **Exception Handling** ซึ่งประกอบด้วย 3 keyword หลัก:

- **`throw`**: "โยน" ข้อผิดพลาดออกไป ทันทีที่ throw ทำงาน โปรแกรมจะหยุดการทำงานปกติทันที
  และเริ่มค้นหา `catch` block ที่รับมือกับ exception นั้นได้
- **`try`**: ครอบโค้ดส่วนที่ "อาจจะ" เกิดข้อผิดพลาด
- **`catch`**: ดักจับและจัดการ exception ที่ถูก throw ออกมาจาก `try` block

```cpp
#include <iostream>

double divide(double a, double b) {
    if (b == 0.0) {
        throw std::string("หารด้วยศูนย์ไม่ได้"); // throw ค่าชนิด std::string
    }
    return a / b;
}

int main() {
    try {
        double r1 = divide(10, 2);
        std::cout << "10 / 2 = " << r1 << '\n';

        double r2 = divide(10, 0); // จะ throw ตรงนี้ ทำให้บรรทัดถัดไปไม่ถูกรันเลย
        std::cout << "10 / 0 = " << r2 << '\n';
        std::cout << "บรรทัดนี้จะไม่ถูกรันเลย\n";
    } catch (const std::string& msg) {
        std::cout << "จับข้อผิดพลาดได้: " << msg << '\n';
    }

    std::cout << "โปรแกรมทำงานต่อได้ตามปกติหลัง catch\n";
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 divide.cpp -o divide
./divide
```

ผลลัพธ์:

```
10 / 2 = 5
จับข้อผิดพลาดได้: หารด้วยศูนย์ไม่ได้
โปรแกรมทำงานต่อได้ตามปกติหลัง catch
```

### สิ่งที่เกิดขึ้นเบื้องหลัง

1. `divide(10, 0)` เรียก `throw std::string(...)`
2. C++ runtime **หยุดการทำงานของฟังก์ชัน `divide` ทันที** (ไม่มีการ `return` ตามปกติ)
3. runtime ค้นหา `catch` block ที่ "จับได้" กับชนิดของสิ่งที่ถูก throw โดยไล่ค้นย้อนกลับไปตาม
   ลำดับการเรียกฟังก์ชัน (call stack) จนกว่าจะเจอ `try` ที่ครอบอยู่และมี `catch` ที่ตรงชนิด
4. เมื่อเจอ `catch (const std::string& msg)` ที่ตรงชนิด โค้ดใน block นั้นจะทำงาน
5. หลังจบ `catch` block โปรแกรม**ทำงานต่อตามปกติ** จากบรรทัดถัดจาก `catch` block ทั้งหมด
   (ไม่ย้อนกลับไปที่จุดที่ throw อีก)

จุดสำคัญที่ต้องเข้าใจ: **`std::cout << "บรรทัดนี้จะไม่ถูกรันเลย\n";` ไม่มีวันถูกรัน** เพราะ
exception ทำให้ควบคุมการทำงานกระโดดออกจาก `try` block ทันทีที่ `throw` เกิดขึ้น ไม่ว่าจะอยู่ลึก
แค่ไหนในฟังก์ชันที่ซ้อนกันกี่ชั้นก็ตาม

---

## 54.2 throw object แทน primitive type (Step 426)

ตัวอย่างข้างบน throw เป็น `std::string` ธรรมดา ซึ่งใช้งานได้แต่มีข้อจำกัด: มันมีแค่ข้อความ
ไม่มีข้อมูลอื่นประกอบ ในทางปฏิบัติเราสามารถ **throw object ของ class ใดก็ได้** ที่เราออกแบบเอง
เพื่อพก **ข้อมูลบริบท (context)** ของข้อผิดพลาดไปด้วย

```cpp
#include <iostream>
#include <string>

class InsufficientFundsError {
public:
    InsufficientFundsError(double balance, double amount)
        : balance_(balance), amount_(amount) {}

    double balance() const { return balance_; }
    double amount() const { return amount_; }

    std::string message() const {
        return "ยอดเงินไม่พอ: มี " + std::to_string(balance_) +
               " แต่ต้องการถอน " + std::to_string(amount_);
    }

private:
    double balance_;
    double amount_;
};

class BankAccount {
public:
    explicit BankAccount(double balance) : balance_(balance) {}

    void withdraw(double amount) {
        if (amount > balance_) {
            throw InsufficientFundsError(balance_, amount); // throw object ไม่ใช่แค่ primitive
        }
        balance_ -= amount;
    }

    double balance() const { return balance_; }

private:
    double balance_;
};

int main() {
    BankAccount acc(1000.0);
    try {
        acc.withdraw(300.0);
        std::cout << "ถอนสำเร็จ เหลือ: " << acc.balance() << '\n';
        acc.withdraw(5000.0); // จะ throw
    } catch (const InsufficientFundsError& e) {
        std::cout << "เกิดข้อผิดพลาด: " << e.message() << '\n';
        std::cout << "  ยอดคงเหลือ: " << e.balance() << ", ที่ต้องการถอน: " << e.amount() << '\n';
    }
}
```

ผลลัพธ์:

```
ถอนสำเร็จ เหลือ: 700
เกิดข้อผิดพลาด: ยอดเงินไม่พอ: มี 700.000000 แต่ต้องการถอน 5000.000000
  ยอดคงเหลือ: 700, ที่ต้องการถอน: 5000
```

สังเกตว่า `InsufficientFundsError` เก็บทั้ง `balance_` และ `amount_` ไว้ ทำให้ผู้ที่ `catch`
สามารถเข้าถึง**ข้อมูลตัวเลขจริง** ไปประมวลผลต่อได้ (เช่น แสดงผลใน UI, log เก็บสถิติ) ไม่ใช่แค่
ข้อความ string ตายตัวเหมือนตัวอย่างก่อนหน้า นี่คือเหตุผลที่โค้ดจริงในโลกอุตสาหกรรมแทบทั้งหมด
**throw object ของ class ที่ออกแบบมาเฉพาะ** แทนที่จะ throw primitive type ตรงๆ

> **หมายเหตุสำคัญ**: การ `throw std::string(...)` ในหัวข้อ 54.1 เป็นตัวอย่างเพื่อให้เห็นกลไก
> พื้นฐานที่สุดของ `throw`/`catch` เท่านั้น ในทางปฏิบัติจริง **ควร throw object ที่สืบทอดจาก
> `std::exception`** เสมอ ซึ่งเราจะเรียนในหัวข้อถัดไป เพราะทำให้เข้ากันได้กับโค้ดส่วนอื่นและ
> library ทั้งหมดที่คาดหวัง `std::exception`

---

## 54.3 std::exception hierarchy (Step 427)

Standard Library มาพร้อมกับ **ลำดับชั้นของ exception class** (hierarchy) ที่ครอบคลุมสถานการณ์
ข้อผิดพลาดทั่วไปอยู่แล้ว ทุกตัวสืบทอดมาจาก `std::exception` (ประกาศใน header `<exception>`
และ `<stdexcept>`) ซึ่งมี virtual member function ที่สำคัญที่สุดคือ `what()` ที่คืนค่า
`const char*` อธิบายข้อผิดพลาด

```
                    std::exception
                          │
        ┌─────────────────┼─────────────────┐
        │                                    │
  std::logic_error                   std::runtime_error
   (ข้อผิดพลาดที่ตรวจจับได้           (ข้อผิดพลาดที่รู้ตอนรันเท่านั้น
    ตั้งแต่ก่อนรันจริง เช่น            เช่น ข้อมูล input จากผู้ใช้ผิดพลาด)
    ข้อผิดพลาดของ logic โปรแกรม)              │
        │                              ┌──────┴──────┐
   ┌────┼────────┐                     │             │
   │    │        │              std::range_error  std::overflow_error
std::invalid_   std::out_of_    (ค่าที่คำนวณได้    (ผลลัพธ์เกินขอบเขตที่
argument         range           อยู่นอกช่วงที่      เก็บได้ เช่น เลข
(argument       (ดัชนี/ตำแหน่ง   ยอมรับได้)         overflow)
ที่ส่งเข้ามา     อยู่นอกขอบเขต
ไม่ถูกต้อง)      เช่น .at() ผิด)
```

```cpp
#include <iostream>
#include <stdexcept>
#include <vector>

int main() {
    std::vector<int> v{1, 2, 3};

    try {
        std::cout << v.at(10) << '\n'; // out_of_range
    } catch (const std::out_of_range& e) {
        std::cout << "out_of_range: " << e.what() << '\n';
    }

    try {
        throw std::invalid_argument("ค่าที่ส่งเข้ามาไม่ถูกต้อง");
    } catch (const std::logic_error& e) {
        // invalid_argument สืบทอดจาก logic_error จึงจับด้วย base class ได้
        std::cout << "logic_error (จริงๆ คือ invalid_argument): " << e.what() << '\n';
    }

    try {
        throw std::runtime_error("เกิดปัญหาระหว่างรันโปรแกรม");
    } catch (const std::exception& e) {
        // exception ทุกตัวใน standard library สืบทอดจาก std::exception
        std::cout << "std::exception: " << e.what() << '\n';
    }
}
```

ผลลัพธ์:

```
out_of_range: vector::_M_range_check: __n (which is 10) >= this->size() (which is 3)
logic_error (จริงๆ คือ invalid_argument): ค่าที่ส่งเข้ามาไม่ถูกต้อง
std::exception: เกิดปัญหาระหว่างรันโปรแกรม
```

> ข้อความจาก `v.at(10)` ในตัวอย่างมาจาก implementation ของ libstdc++ (GCC) โดยตรง คอมไพเลอร์/
> library อื่นอาจแสดงข้อความต่างกันเล็กน้อย แต่ชนิดของ exception (`std::out_of_range`) จะเหมือน
> กันเสมอตามมาตรฐาน

### exception มาตรฐานที่ใช้บ่อยที่สุด

| Exception | อยู่ใน header | ใช้เมื่อ |
|---|---|---|
| `std::logic_error` | `<stdexcept>` | ข้อผิดพลาดที่ควรตรวจจับได้ตั้งแต่ก่อนรันจริง (บั๊กของโปรแกรมเมอร์) |
| `std::invalid_argument` | `<stdexcept>` | argument ที่รับเข้ามาไม่ถูกต้องตามที่ฟังก์ชันคาดหวัง |
| `std::out_of_range` | `<stdexcept>` | เข้าถึงตำแหน่ง/ดัชนีที่อยู่นอกขอบเขตที่ยอมรับได้ |
| `std::runtime_error` | `<stdexcept>` | ข้อผิดพลาดที่รู้ได้เฉพาะตอนรันโปรแกรมจริงเท่านั้น |
| `std::bad_alloc` | `<new>` | จองหน่วยความจำด้วย `new` ไม่สำเร็จ (memory หมด) |
| `std::bad_cast` | `<typeinfo>` | `dynamic_cast` แบบ reference ล้มเหลว (จะเรียนใน Module E/F) |

---

## 54.4 เขียน custom exception class สืบทอดจาก std::exception (Step 428)

การ throw class ของตัวเองอย่างใน 54.2 ใช้งานได้ แต่ยังมีข้อเสียคือ **ไม่เข้ากันกับโค้ดที่
`catch (const std::exception& e)`** เพราะ `InsufficientFundsError` ไม่ได้สืบทอดจาก
`std::exception` เลย ในทางปฏิบัติที่ถูกต้อง เราควรออกแบบ custom exception ให้สืบทอดจาก
`std::exception` (หรือ class ลูกที่เหมาะสมกว่า เช่น `std::runtime_error`) เสมอ

มีสองแนวทางหลัก:

**แนวทางที่ 1**: สืบทอดจาก `std::runtime_error` แล้วส่งข้อความผ่าน constructor ของมันตรงๆ
(ง่ายและพบบ่อยที่สุดในโค้ดจริง)

**แนวทางที่ 2**: สืบทอดจาก `std::exception` โดยตรง และ override `what()` เอง (ใช้เมื่อต้องการ
ควบคุมข้อความแบบ dynamic เต็มรูปแบบ)

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

class BookNotFoundError : public std::runtime_error {
public:
    explicit BookNotFoundError(const std::string& isbn)
        : std::runtime_error("ไม่พบหนังสือ ISBN: " + isbn), isbn_(isbn) {}

    const std::string& isbn() const noexcept { return isbn_; }

private:
    std::string isbn_;
};

// custom exception ที่ override what() เอง แทนที่จะพึ่ง runtime_error
class OutOfStockError : public std::exception {
public:
    explicit OutOfStockError(std::string title) : title_(std::move(title)) {
        message_ = "หนังสือ \"" + title_ + "\" หมดสต๊อก";
    }

    const char* what() const noexcept override {
        return message_.c_str();
    }

    const std::string& title() const noexcept { return title_; }

private:
    std::string title_;
    std::string message_;
};

void findBook(const std::string& isbn) {
    if (isbn != "978-0-13-110362-7") {
        throw BookNotFoundError(isbn);
    }
}

void checkStock(const std::string& title, int stock) {
    if (stock <= 0) {
        throw OutOfStockError(title);
    }
}

int main() {
    try {
        findBook("999-9-99-999999-9");
    } catch (const BookNotFoundError& e) {
        std::cout << "จับได้ (custom, runtime_error-based): " << e.what()
                  << " (isbn=" << e.isbn() << ")\n";
    }

    try {
        checkStock("The C Programming Language", 0);
    } catch (const OutOfStockError& e) {
        std::cout << "จับได้ (custom, std::exception-based): " << e.what() << '\n';
    }

    // ทั้งสอง custom exception ถูกจับได้ด้วย std::exception& เช่นกัน เพราะสืบทอดมา
    try {
        findBook("111-1-11-111111-1");
    } catch (const std::exception& e) {
        std::cout << "จับผ่าน std::exception&: " << e.what() << '\n';
    }
}
```

ผลลัพธ์:

```
จับได้ (custom, runtime_error-based): ไม่พบหนังสือ ISBN: 999-9-99-999999-9 (isbn=999-9-99-999999-9)
จับได้ (custom, std::exception-based): หนังสือ "The C Programming Language" หมดสต๊อก
จับผ่าน std::exception&: ไม่พบหนังสือ ISBN: 111-1-11-111111-1
```

### จุดสำคัญที่ต้องระวังตอน override `what()`

- `what()` ต้องมี signature ตรงเป๊ะ: `const char* what() const noexcept override` — ทั้ง
  `const` (หลังชื่อฟังก์ชัน) และ `noexcept` ต้องตรงกับ base class มิฉะนั้นจะไม่ใช่การ override
  จริง (compiler จะเตือนถ้าใช้ `override` แล้วไม่ตรง)
- `what()` ต้องคืนค่า pointer ที่ **ยังมีอายุอยู่หลังจากฟังก์ชันจบการทำงาน** ห้ามคืน pointer ไป
  ยัง local variable ที่ตายไปแล้ว (เช่น `std::string` ที่สร้างในฟังก์ชันแล้ว `.c_str()` แล้ว
  return ออกไป — เป็น undefined behavior ทันที) วิธีที่ปลอดภัยคือเก็บ `std::string message_`
  ไว้เป็น member ของ class ตามตัวอย่างข้างบน แล้วคืน `.c_str()` ของ member ตัวนั้น เพราะ member
  จะมีอายุเท่ากับ object

---

## 54.5 catch หลาย type และลำดับความสำคัญ (Step 429)

เราสามารถเขียน `catch` หลาย block ต่อกันเพื่อจัดการ exception หลายชนิดแตกต่างกัน แต่มี **กฎ
สำคัญมาก**: exception ชนิดที่ **เฉพาะเจาะจงกว่า (derived class) ต้องมาก่อน** ชนิดที่ทั่วไปกว่า
(base class) เสมอ เพราะ C++ จะ**เลือก `catch` block แรกที่ตรงกัน** จากบนลงล่าง แล้วไม่มองตัว
ถัดไปเลย

```cpp
#include <iostream>
#include <stdexcept>

void riskyOperation(int code) {
    if (code == 1) throw std::out_of_range("out_of_range เกิดขึ้น");
    if (code == 2) throw std::runtime_error("runtime_error เกิดขึ้น");
    if (code == 3) throw std::string("string ธรรมดา");
    if (code == 4) throw 42;
    std::cout << "ไม่มีข้อผิดพลาด\n";
}

void handle(int code) {
    try {
        riskyOperation(code);
    } catch (const std::out_of_range& e) {
        // ต้องมาก่อน std::exception เพราะเจาะจงกว่า (derived class)
        std::cout << "จับเฉพาะ out_of_range: " << e.what() << '\n';
    } catch (const std::exception& e) {
        // จับ std::exception ทุกชนิดที่เหลือ (ยกเว้น out_of_range ที่จับไปแล้วด้านบน)
        std::cout << "จับ std::exception ทั่วไป: " << e.what() << '\n';
    } catch (...) {
        // จับทุกอย่างที่เหลือ ไม่ว่าจะเป็นชนิดใดก็ตาม (string, int, ฯลฯ)
        std::cout << "จับด้วย catch(...) ไม่ทราบชนิดที่แน่ชัด\n";
    }
}

int main() {
    for (int code = 1; code <= 4; ++code) {
        handle(code);
    }
}
```

ผลลัพธ์:

```
จับเฉพาะ out_of_range: out_of_range เกิดขึ้น
จับ std::exception ทั่วไป: runtime_error เกิดขึ้น
จับด้วย catch(...) ไม่ทราบชนิดที่แน่ชัด
จับด้วย catch(...) ไม่ทราบชนิดที่แน่ชัด
```

สังเกตว่า `code == 1` (`out_of_range`) และ `code == 2` (`runtime_error`) ต่างก็สืบทอดจาก
`std::exception` แต่ `out_of_range` ถูกจับด้วย `catch` block แรกเพราะเจาะจงกว่า ส่วน
`code == 3` และ `code == 4` (throw `std::string` และ `int` ตรงๆ ซึ่ง**ไม่ใช่**
`std::exception`) ไม่มี `catch` ไหนตรงชนิดเลย จึงตกไปที่ `catch(...)` ซึ่งเป็น**ตัวจับ
ทุกสิ่งที่เหลือ**

### ถ้าเขียนลำดับผิด (base class มาก่อน derived class)

ลองสลับลำดับ `catch` ดู (ตัวอย่างนี้จงใจเขียนผิดเพื่อสาธิต):

```cpp
#include <iostream>
#include <stdexcept>

int main() {
    try {
        throw std::out_of_range("test");
    } catch (const std::exception& e) {
        std::cout << "base: " << e.what() << '\n';
    } catch (const std::out_of_range& e) {
        std::cout << "derived: " << e.what() << '\n';
    }
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 catch_order_broken.cpp -o catch_order_broken
```

**GCC ฉลาดพอที่จะเตือนบั๊กนี้ให้ตอน compile เลย** (นี่คือเหตุผลสำคัญที่ต้องเปิด `-Wall
-Wextra` เสมอ):

```
catch_order_broken.cpp: In function 'int main()':
catch_order_broken.cpp:9:7: warning: exception of type 'std::out_of_range' will be caught by earlier handler [-Wexceptions]
    9 |     } catch (const std::out_of_range& e) {
      |       ^~~~~
catch_order_broken.cpp:7:7: note: for type 'std::exception'
    7 |     } catch (const std::exception& e) {
      |       ^~~~~
```

ถ้ารันโปรแกรมนี้ (แม้จะ compile ผ่านพร้อม warning) ผลลัพธ์จะเป็น `base: test` เสมอ เพราะ
`catch (const std::exception& e)` จับได้ก่อนและ C++ จะไม่มองต่อไปที่ `catch` block ที่สอง
เลย ทำให้ `catch (const std::out_of_range& e)` เป็น**โค้ดที่ไม่มีวันถูกเรียกใช้ (dead code)**

### catch(...) — จับทุกสิ่งที่เหลือ

`catch (...)` (มี `...` เป็นชนิด) คือ catch-all handler ที่จับ **ทุก exception ทุกชนิด** ไม่ว่า
จะเป็น `std::exception`, `std::string`, `int`, หรือ object ของ class ใดก็ตาม ข้อจำกัดคือ
**ไม่สามารถเข้าถึงข้อมูลของสิ่งที่ถูก throw ได้เลย** (ไม่รู้ชนิด ไม่มีตัวแปรให้ใช้) จึงเหมาะกับ
การใช้เป็น **"ตาข่ายนิรภัยชั้นสุดท้าย"** เพื่อป้องกันโปรแกรม crash จาก exception ที่ไม่คาดคิด
มากกว่าจะใช้เป็นวิธีจัดการ error หลักของระบบ และ**ต้องอยู่เป็น `catch` block สุดท้ายเสมอ**
(เพราะมันจับได้ทุกอย่าง ถ้าวางไว้ก่อนจะทำให้ `catch` block อื่นที่ตามมาเป็น dead code ทั้งหมด)

---

## 54.6 RAII กับ Exception Safety (Stack Unwinding) (Step 430)

นี่คือหัวข้อที่สำคัญที่สุดของ Part นี้ และเป็นเหตุผลที่แท้จริงว่าทำไม **RAII** (Resource
Acquisition Is Initialization ที่เรียนใน Part 46) ถึงเป็นหัวใจของการเขียน C++ ที่ปลอดภัย

### Stack Unwinding คืออะไร

เมื่อเกิด `throw` ขึ้น และยังไม่มี `catch` ที่ตรงในฟังก์ชันปัจจุบัน C++ runtime จะ**ทำลาย
(destroy) local object ทั้งหมดในฟังก์ชันนั้นตามลำดับย้อนกลับ** (เหมือนออกจาก scope ปกติ) ก่อน
จะ "ปีน" กลับไปหาฟังก์ชันที่เรียกมันมา (caller) แล้วทำซ้ำแบบเดียวกันไปเรื่อยๆ จนกว่าจะเจอ `try`
ที่มี `catch` ตรงชนิด กระบวนการนี้เรียกว่า **Stack Unwinding**

จุดสำคัญคือ: **destructor ของทุก local object จะถูกเรียกโดยอัตโนมัติระหว่าง unwinding นี้เสมอ**
ไม่ว่าฟังก์ชันจะซ้อนกันลึกแค่ไหน — นี่คือเหตุผลที่ถ้าเราผูก resource (memory, file handle,
mutex, network connection) ไว้กับ**อายุของ object** ตามหลัก RAII แล้ว **resource นั้นจะถูก
คืนอัตโนมัติเสมอ แม้เกิด exception ขึ้นกลางทาง**

```cpp
#include <iostream>
#include <memory>
#include <stdexcept>

class FileHandle {
public:
    explicit FileHandle(std::string name) : name_(std::move(name)) {
        std::cout << "  [เปิดไฟล์] " << name_ << '\n';
    }
    ~FileHandle() {
        std::cout << "  [ปิดไฟล์อัตโนมัติ] " << name_ << '\n'; // รันเสมอแม้ exception unwind ผ่าน
    }
private:
    std::string name_;
};

void processRecord(int id) {
    FileHandle log("log_" + std::to_string(id) + ".txt"); // RAII: ผูกทรัพยากรกับ scope
    auto buffer = std::make_unique<int[]>(4);              // ทรัพยากร heap ที่ต้องคืน

    std::cout << "  กำลังประมวลผล record " << id << '\n';
    if (id == 2) {
        throw std::runtime_error("record ที่ 2 มีข้อมูลเสีย");
    }
    std::cout << "  ประมวลผล record " << id << " สำเร็จ\n";
    // จบ scope ปกติ: FileHandle::~FileHandle() และ unique_ptr ทำงานให้อัตโนมัติ
}

int main() {
    std::cout << "เริ่มประมวลผลทั้งหมด\n";
    try {
        for (int id = 1; id <= 3; ++id) {
            std::cout << "-- record " << id << " --\n";
            processRecord(id);
        }
    } catch (const std::runtime_error& e) {
        // ระหว่าง stack unwinding จาก processRecord(2) กลับมาที่นี่
        // destructor ของ FileHandle และ unique_ptr ถูกเรียกให้โดยอัตโนมัติแล้ว
        // ก่อนที่ catch block นี้จะทำงานด้วยซ้ำ -- นี่คือหัวใจของ exception safety ด้วย RAII
        std::cout << "จับข้อผิดพลาดได้ที่ main: " << e.what() << '\n';
    }
    std::cout << "โปรแกรมจบการทำงานโดยไม่มี resource รั่วไหลเลย\n";
}
```

ผลลัพธ์:

```
เริ่มประมวลผลทั้งหมด
-- record 1 --
  [เปิดไฟล์] log_1.txt
  กำลังประมวลผล record 1
  ประมวลผล record 1 สำเร็จ
  [ปิดไฟล์อัตโนมัติ] log_1.txt
-- record 2 --
  [เปิดไฟล์] log_2.txt
  กำลังประมวลผล record 2
  [ปิดไฟล์อัตโนมัติ] log_2.txt
จับข้อผิดพลาดได้ที่ main: record ที่ 2 มีข้อมูลเสีย
โปรแกรมจบการทำงานโดยไม่มี resource รั่วไหลเลย
```

สังเกตบรรทัด **`[ปิดไฟล์อัตโนมัติ] log_2.txt`** — มันถูกพิมพ์**ก่อน**ที่ `catch` block ใน
`main` จะทำงานเสียอีก! นี่คือหลักฐานชัดเจนว่า destructor ของ `FileHandle` ถูกเรียกระหว่าง
stack unwinding โดยอัตโนมัติ ก่อนที่ control flow จะไปถึง `catch` เลยด้วยซ้ำ และสังเกตว่า
`buffer` (ที่เป็น `std::unique_ptr`) ก็ถูกคืน memory ให้อัตโนมัติเช่นกันโดยไม่ต้องเขียน
`delete[]` เอง — ถ้าเราใช้ raw pointer + `new[]`/`delete[]` แบบ manual แทน `unique_ptr`
(สมมติว่าลืม try-finally หรือไม่มี finally ใน C++) memory ก้อนนี้จะ **รั่วไหล (leak) ทันที**
เพราะโค้ดที่ควรจะ `delete[] buffer;` (ถ้าเขียนไว้ท้ายฟังก์ชัน) ไม่มีวันถูกรันเลยเมื่อเกิด
exception ก่อนถึงบรรทัดนั้น

### เปรียบเทียบ: ไม่มี RAII vs มี RAII เมื่อเกิด exception

| สถานการณ์ | ไม่ใช้ RAII (raw pointer + manual cleanup) | ใช้ RAII (smart pointer / wrapper class) |
|---|---|---|
| ทำงานสำเร็จปกติ | ต้องจำ `delete`/`fclose`/`unlock` เองท้ายฟังก์ชัน | destructor คืนทรัพยากรอัตโนมัติเมื่อจบ scope |
| เกิด exception กลางทาง | โค้ด cleanup ท้ายฟังก์ชัน **ไม่ถูกรันเลย** → resource รั่วไหล | destructor ถูกเรียกโดย stack unwinding เสมอ ไม่ว่าจะออกจาก scope แบบใด |
| ต้องเขียน try/catch ครอบทุกจุดที่มี resource หรือไม่ | ต้อง (และเขียนถูกยากมากถ้ามีหลาย resource) | ไม่ต้อง — ผูก resource กับ object แทน |

นี่คือเหตุผลที่ **C++ Core Guidelines** (แนวปฏิบัติที่ดีที่สุดของภาษา ซึ่งจะพูดถึงอย่างละเอียด
ใน Part 80) ยึดหลัก **"RAII ทุกที่ที่ทำได้"** เป็นกฎข้อแรกๆ ของการจัดการ resource อย่างปลอดภัย
ร่วมกับ exception — เรื่อง smart pointer แบบเจาะลึก (`unique_ptr`, `shared_ptr`, `weak_ptr`)
จะเรียนเต็มรูปแบบใน **Part 67** และ RAII pattern ระดับ production จะเรียนใน **Part 68**

---

## 54.7 noexcept keyword เบื้องต้น (Step 431)

`noexcept` คือ keyword ที่บอก compiler (และผู้อ่านโค้ด) ว่า **"ฟังก์ชันนี้รับประกันว่าจะไม่ throw
exception ออกมาเด็ดขาด"**

```cpp
#include <iostream>
#include <type_traits>
#include <vector>

// noexcept บอก compiler และผู้ใช้ว่าฟังก์ชันนี้จะไม่ throw exception ออกมาเด็ดขาด
double square(double x) noexcept {
    return x * x;
}

class Point {
public:
    Point(double x, double y) noexcept : x_(x), y_(y) {}

    // ทำเครื่องหมาย move constructor ว่า noexcept ทำให้ std::vector ยอมใช้ move
    // แทนการ copy ตอน resize (ถ้าไม่ noexcept vector จะเลือก copy เพื่อความปลอดภัย)
    Point(Point&& other) noexcept : x_(other.x_), y_(other.y_) {}

    double x() const noexcept { return x_; }
    double y() const noexcept { return y_; }

private:
    double x_;
    double y_;
};

int main() {
    std::cout << "square(5) = " << square(5) << '\n';
    std::cout << std::boolalpha;
    std::cout << "square เป็น noexcept หรือไม่: " << noexcept(square(5)) << '\n';
    std::cout << "Point move ctor เป็น noexcept หรือไม่: "
              << std::is_nothrow_move_constructible<Point>::value << '\n';

    std::vector<Point> points;
    points.emplace_back(1.0, 2.0);
    points.emplace_back(3.0, 4.0);
    for (const auto& p : points) {
        std::cout << "(" << p.x() << ", " << p.y() << ")\n";
    }
}
```

ผลลัพธ์:

```
square(5) = 25
square เป็น noexcept หรือไม่: true
Point move ctor เป็น noexcept หรือไม่: true
(1, 2)
(3, 4)
```

### ทำไม noexcept สำคัญ

1. **สื่อสาร contract ที่ชัดเจน**: ผู้เรียกฟังก์ชันรู้ทันทีว่าไม่จำเป็นต้องเขียน `try/catch`
   ครอบเพื่อรับมือ exception จากฟังก์ชันนี้
2. **มีผลต่อการตัดสินใจของ Standard Library**: อย่างในตัวอย่าง `std::vector` เมื่อต้อง
   resize (ขยายขนาด array ภายใน) มันต้องย้าย object เดิมทั้งหมดไปที่หน่วยความจำก้อนใหม่
   ถ้า move constructor ของ `Point` **ไม่ได้** ทำเครื่องหมาย `noexcept` ไว้ `vector` จะเลือก
   ใช้ **copy constructor แทน move constructor** เพื่อความปลอดภัย (เพราะถ้า move
   constructor throw กลางทาง ข้อมูลอาจเสียหายแบบกู้คืนไม่ได้) การใส่ `noexcept` ให้ move
   constructor จึงช่วยเพิ่มประสิทธิภาพได้จริงในโค้ดที่ใช้ container เยอะๆ (เรื่อง Move
   Semantics แบบเจาะลึกจะเรียนใน **Part 70**)
3. **operator `noexcept(expr)`**: ใช้ตรวจสอบตอน compile-time ว่านิพจน์หนึ่งถูกประกาศว่า
   `noexcept` หรือไม่ (คืนค่า `bool`) มีประโยชน์มากตอนเขียน template ขั้นสูง

> **ข้อควรระวัง**: `noexcept` เป็นแค่ "คำสัญญา" ไม่ใช่การบังคับของ compiler ถ้าฟังก์ชันที่
> ประกาศ `noexcept` ดัน throw exception ออกมาจริงๆ ระหว่างรัน โปรแกรมจะเรียก
> **`std::terminate()`** ทันทีและปิดตัวลงแบบไม่มีการ unwind stack ตามปกติ — ดังนั้นห้ามใส่
> `noexcept` ให้ฟังก์ชันที่มีโอกาส throw จริงๆ เด็ดขาด

---

## 54.8 ข้อควรระวัง: throw ใน destructor (Step 432)

นี่คือกฎที่**เข้มงวดที่สุดข้อหนึ่ง**ของ Exception Handling ใน C++: **ห้าม throw exception ออก
จาก destructor เด็ดขาด**

### เหตุผลเชิงเทคนิค

ตั้งแต่ C++11 เป็นต้นไป destructor ทุกตัวถูก compiler ทำเครื่องหมายเป็น **`noexcept` โดย
default โดยอัตโนมัติ** (เพราะเหตุผลด้านล่างนี้) ถ้า destructor throw exception ออกมาจริง จะเกิด
สถานการณ์ที่เรียกว่า **"double exception"** — เช่น ถ้าโปรแกรมกำลัง unwind stack เพราะมี
exception ตัวหนึ่งเกิดขึ้นอยู่แล้ว แล้วระหว่าง unwinding นั้น destructor ของ local object ตัว
หนึ่งดัน throw exception ตัวที่สองซ้อนขึ้นมาอีก C++ **ไม่รู้ว่าจะจัดการ exception ไหนก่อน** จึง
เรียก **`std::terminate()`** ทันทีเพื่อปิดโปรแกรมแบบฉุกเฉิน (ไม่มีโอกาสได้ `catch` เลย)

```cpp
#include <iostream>
#include <stdexcept>

class Dangerous {
public:
    ~Dangerous() noexcept(false) {   // ต้องปิด noexcept ที่ compiler ใส่ให้ destructor โดย default
        std::cout << "destructor กำลังทำงาน...\n";
        throw std::runtime_error("throw จาก destructor!"); // อันตรายมาก
    }
};

int main() {
    try {
        Dangerous d1;
        Dangerous d2;
        throw std::runtime_error("exception แรกจาก main");
        // ระหว่าง stack unwinding จาก exception แรก d2 และ d1 จะถูกทำลาย
        // ถ้า destructor ของ d2 throw ซ้ำระหว่างที่ exception แรกยังไม่ถูกจัดการ
        // จะเกิด exception สองตัวพร้อมกัน -> C++ เรียก std::terminate() ทันที
    } catch (const std::exception& e) {
        std::cout << "จับได้: " << e.what() << '\n'; // โค้ดส่วนนี้จะไม่มีวันถูกรัน
    }
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 dangerous.cpp -o dangerous
./dangerous
```

ผลลัพธ์:

```
destructor กำลังทำงาน...
terminate called after throwing an instance of 'std::runtime_error'
  what():  throw จาก destructor!
Aborted (core dumped)
```

โปรแกรม**ถูกปิดฉุกเฉินทันที** (`Aborted`) โดยที่ `catch` block ใน `main` **ไม่มีวันถูกรันเลย**
แม้จะเขียน `try/catch` ครอบไว้ถูกต้องตามหลักการทุกอย่างก็ตาม — นี่คือเหตุผลที่ต้องจำกฎนี้ให้
ขึ้นใจ: **destructor ต้องไม่ throw exception เด็ดขาด**

### ควรทำอย่างไรแทน

ถ้า destructor ต้องทำงานที่มีโอกาสล้มเหลว (เช่น ปิดไฟล์ ปิด network connection) ให้ใช้แนวทาง
เหล่านี้แทนการ throw:

1. **จับ exception ไว้ภายใน destructor เอง** ด้วย `try/catch` แล้ว log ข้อผิดพลาดไว้แทนที่จะ
   ปล่อยให้ throw ออกไป
2. **ให้ผู้ใช้เรียก method แยกต่างหาก** (เช่น `close()`) ก่อนที่ object จะถูกทำลาย เพื่อให้มี
   โอกาส throw และจัดการ error ได้ตามปกติ แล้วให้ destructor แค่ทำ cleanup แบบเงียบๆ (silent
   best-effort) เป็นตาข่ายนิรภัยสุดท้ายเท่านั้น
3. **ทำเครื่องหมายว่า operation ที่เสี่ยงจะ throw นั้น "ไม่ควรเกิดขึ้นถ้าใช้งานถูกต้อง"** และถ้า
   เกิดขึ้นจริงให้ยอมรับว่าเป็นบั๊กร้ายแรงที่ควร `std::terminate()` ไปเลย (สำหรับกรณีที่ error
   นั้นกู้คืนไม่ได้จริงๆ)

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **throw primitive type หรือ std::string ตรงๆ แทนที่จะ throw class ที่สืบทอดจาก
   `std::exception`** — ทำให้โค้ดส่วนอื่นที่คาดหวัง `catch (const std::exception&)` จับไม่ได้
   เลย ควรออกแบบ custom exception ให้สืบทอดจาก `std::exception` หรือลูกของมันเสมอ
2. **catch ตาม base class ก่อน derived class** — ทำให้ catch block ของ derived class กลาย
   เป็น dead code ที่ไม่มีวันถูกเรียก (GCC เตือนด้วย `-Wexceptions` แต่ถ้าไม่เปิด warning จะ
   ไม่รู้ตัวเลย)
3. **catch by value แทนที่จะ catch by reference** (`catch (std::exception e)` แทน
   `catch (const std::exception& e)`) — ทำให้เกิด **object slicing** เหมือนที่เรียนใน Part 49
   เรื่อง Polymorphism: ถ้า exception จริงเป็น derived class แต่ catch by value ของ base
   class จะถูก "ตัด" ข้อมูลส่วนที่เป็น derived ทิ้งไปหมด และ virtual function เช่น `what()`
   อาจแสดงผลผิดจากที่ควรจะเป็น ควร **`catch by const reference` เสมอ**
4. **ใช้ exception สำหรับ control flow ปกติที่ไม่ใช่ข้อผิดพลาดจริง** — เช่น ใช้ throw เพื่อ
   ออกจาก loop ซ้อนกันหลายชั้นแทนการออกแบบโค้ดให้ดีขึ้น exception ควรสงวนไว้สำหรับสถานการณ์ที่
   "ผิดปกติจริง" เท่านั้น เพราะการ throw/catch มี cost ด้าน performance สูงกว่าการ return
   ค่าปกติมาก
5. **คืน pointer/reference ไปยัง local variable ใน `what()`** — ทำให้เกิด undefined behavior
   เพราะ local variable ถูกทำลายไปแล้วตั้งแต่ `what()` return ทบทวนหัวข้อ 54.4 ต้องเก็บ
   ข้อความไว้เป็น member ของ exception class เสมอ
6. **throw exception ออกจาก destructor** — จุดที่อันตรายที่สุด เพราะทำให้เกิด
   `std::terminate()` ทันทีถ้าเกิดขึ้นระหว่าง stack unwinding ของ exception อีกตัวหนึ่งอยู่แล้ว
   (ทบทวนหัวข้อ 54.8 อย่างละเอียด)
7. **ลืมว่า `noexcept` เป็นแค่คำสัญญา ไม่ใช่การบังคับ** — ถ้าฟังก์ชันที่ประกาศ `noexcept`
   throw จริง โปรแกรมจะ `std::terminate()` ทันที ไม่ต่างจากกรณี destructor เลย

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `parseAge(const std::string& text)` ที่แปลง string เป็นอายุ (int) โดย
   throw `std::invalid_argument` ถ้า string ไม่ใช่ตัวเลข และ throw `std::out_of_range` ถ้า
   อายุที่ได้อยู่นอกช่วง 0-150
2. เขียน custom exception class ชื่อ `NegativeValueError` ที่สืบทอดจาก `std::invalid_argument`
   พร้อมเก็บค่าตัวเลขที่ผิดพลาดไว้เป็น member และมี getter คืนค่านั้น
3. เขียนฟังก์ชันที่มี `try`/`catch` หลายชั้นซ้อนกัน (nested try-catch) ทดสอบว่าลำดับการจับ
   catch ทำงานถูกต้องตามที่คาดหวัง โดยจงใจให้มีทั้ง exception ที่จับได้และจับไม่ได้ (ตกไปที่
   `catch(...)`)
4. เขียน class `ScopedTimer` ที่จับเวลาตั้งแต่สร้าง object จนถูกทำลาย แล้วพิมพ์เวลาที่ใช้ไปออก
   มาตอน destructor ทำงาน ทดสอบว่ามันยังคงพิมพ์เวลาออกมาถูกต้องแม้ scope ที่มันอยู่จะออกเพราะ
   เกิด exception ก็ตาม (สาธิตหลักการ RAII/stack unwinding)
5. ทดลองเขียนฟังก์ชันสองแบบ แบบหนึ่งมี `noexcept` อีกแบบไม่มี แล้วใช้
   `std::is_nothrow_move_constructible` พิสูจน์ว่า `noexcept` มีผลต่อพฤติกรรมของ
   `std::vector` เวลา resize จริงหรือไม่
6. อภิปรายและยกตัวอย่างสถานการณ์จริง (ไม่ต้องเขียนโค้ดที่รันจริงก็ได้) ที่การ throw exception
   ใน destructor จะทำให้เกิด `std::terminate()` และอธิบายว่าควรออกแบบใหม่อย่างไรเพื่อหลีกเลี่ยง
   ปัญหานี้

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

int parseAge(const std::string& text) {
    std::size_t pos = 0;
    int age = 0;
    try {
        age = std::stoi(text, &pos);
    } catch (const std::invalid_argument&) {
        throw std::invalid_argument("\"" + text + "\" ไม่ใช่ตัวเลข");
    }

    if (pos != text.size()) {
        throw std::invalid_argument("\"" + text + "\" มีอักขระที่ไม่ใช่ตัวเลขปะปนอยู่");
    }
    if (age < 0 || age > 150) {
        throw std::out_of_range("อายุ " + std::to_string(age) + " อยู่นอกช่วงที่เป็นไปได้ (0-150)");
    }
    return age;
}

void tryParse(const std::string& text) {
    try {
        int age = parseAge(text);
        std::cout << "\"" << text << "\" -> อายุ: " << age << '\n';
    } catch (const std::out_of_range& e) {
        std::cout << "\"" << text << "\" -> out_of_range: " << e.what() << '\n';
    } catch (const std::invalid_argument& e) {
        std::cout << "\"" << text << "\" -> invalid_argument: " << e.what() << '\n';
    }
}

int main() {
    tryParse("25");
    tryParse("abc");
    tryParse("25x");
    tryParse("999");
    tryParse("-5");
}
```

ผลลัพธ์:

```
"25" -> อายุ: 25
"abc" -> invalid_argument: "abc" ไม่ใช่ตัวเลข
"25x" -> invalid_argument: "25x" มีอักขระที่ไม่ใช่ตัวเลขปะปนอยู่
"999" -> out_of_range: อายุ 999 อยู่นอกช่วงที่เป็นไปได้ (0-150)
"-5" -> out_of_range: อายุ -5 อยู่นอกช่วงที่เป็นไปได้ (0-150)
```

จุดสำคัญ: ฟังก์ชันนี้จับ `std::invalid_argument` ที่ `std::stoi` อาจ throw (เมื่อ string ไม่ใช่
ตัวเลขเลย เช่น `"abc"`) แล้ว **throw ใหม่อีกครั้งด้วยข้อความที่เข้าใจง่ายกว่า** เทคนิคนี้เรียก
ว่า **exception translation** — จับ exception ระดับล่างที่ implementation-specific แล้วแปลงเป็น
exception ระดับที่เหมาะกับ business logic ของเรามากกว่า ก่อนส่งต่อออกไปให้ผู้เรียกใช้งานจริง

### แนวทางเฉลยข้อ 4

```cpp
#include <chrono>
#include <iostream>
#include <stdexcept>
#include <string>

class ScopedTimer {
public:
    explicit ScopedTimer(std::string label)
        : label_(std::move(label)), start_(std::chrono::steady_clock::now()) {}

    // destructor รันเสมอไม่ว่าจะออกจาก scope ตามปกติหรือเพราะ exception (stack unwinding)
    ~ScopedTimer() {
        auto end = std::chrono::steady_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(end - start_).count();
        std::cout << "[" << label_ << "] ใช้เวลา " << us << " microseconds\n";
    }

private:
    std::string label_;
    std::chrono::steady_clock::time_point start_;
};

volatile long sinkValue = 0; // ป้องกัน compiler optimize loop ทิ้งไป

void busyWork(long iterations) {
    long total = 0;
    for (long i = 0; i < iterations; ++i) {
        total += i % 7;
    }
    sinkValue = total;
}

void doWork(bool shouldFail) {
    ScopedTimer timer("doWork");
    busyWork(2'000'000);
    if (shouldFail) {
        throw std::runtime_error("งานล้มเหลวระหว่างทำ");
    }
    std::cout << "doWork ทำงานสำเร็จ\n";
}

int main() {
    try {
        doWork(false);
        doWork(true);
    } catch (const std::exception& e) {
        // ScopedTimer ของ doWork(true) ถูกทำลาย (พิมพ์เวลาออกมา) ไปแล้ว ก่อนจะมาถึง catch ตรงนี้
        std::cout << "จับข้อผิดพลาดได้: " << e.what() << '\n';
    }
}
```

ตัวอย่างผลลัพธ์ (ตัวเลข microseconds จริงจะต่างกันไปตามเครื่อง):

```
doWork ทำงานสำเร็จ
[doWork] ใช้เวลา 5313 microseconds
[doWork] ใช้เวลา 5285 microseconds
จับข้อผิดพลาดได้: งานล้มเหลวระหว่างทำ
```

จุดสำคัญ: การเรียกครั้งที่สอง (`doWork(true)`) throw exception กลางฟังก์ชัน แต่ `ScopedTimer`
**ยังคงพิมพ์เวลาที่ใช้ออกมาได้ถูกต้อง** ก่อนที่ `catch` ใน `main` จะทำงานเสียอีก เพราะ
destructor ของมันถูกเรียกโดยกลไก stack unwinding โดยอัตโนมัติ — พิสูจน์ให้เห็นชัดเจนว่า RAII
ทำงานได้แม้ scope นั้นจะออกด้วยเหตุผลผิดปกติ (exception) ไม่ใช่แค่การ return ตามปกติเท่านั้น

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้:

- กลไก **`try`/`catch`/`throw`** ที่เป็นรากฐานของการจัดการข้อผิดพลาดใน C++ ซึ่งดีกว่าการคืน
  error code แบบภาษา C ตรงที่ผู้เขียนโค้ดไม่สามารถ "ลืม" ตรวจสอบข้อผิดพลาดได้ง่ายเหมือนเดิม
- การ **throw object** ที่พกข้อมูลบริบทไปด้วย แทนที่จะ throw แค่ primitive type
- โครงสร้าง **std::exception hierarchy** และ exception มาตรฐานที่ใช้บ่อยที่สุด
  (`runtime_error`, `logic_error`, `out_of_range`, `invalid_argument`)
- การเขียน **custom exception class** ที่สืบทอดจาก `std::exception` อย่างถูกต้องและปลอดภัย
  พร้อม override `what()`
- กฎการเรียงลำดับ **`catch`** จากเฉพาะเจาะจงไปทั่วไป และการใช้ **`catch(...)`** เป็นตาข่าย
  นิรภัยชั้นสุดท้าย
- ทำไม **RAII** และ **Stack Unwinding** คือกลไกที่ทำให้ C++ จัดการ resource ได้ปลอดภัยแม้เกิด
  exception ขึ้นกลางทาง
- การใช้ **`noexcept`** เพื่อสื่อสาร contract และเพิ่มประสิทธิภาพให้กับ Standard Library
- ข้อห้ามที่สำคัญที่สุด: **ห้าม throw exception ออกจาก destructor เด็ดขาด** เพราะจะนำไปสู่
  `std::terminate()` ทันที

ตอนนี้เรามีองค์ประกอบครบทุกชิ้นของ OOP ใน C++ แล้ว — Class, Constructor/Destructor,
Encapsulation, Inheritance, Polymorphism, Abstract Class, Operator Overloading, Friend, Static
Member, และ Exception Handling ใน **Part 55** ซึ่งเป็น Part สุดท้ายของ Module D เราจะนำทุก
แนวคิดเหล่านี้มา**ประกอบร่างเป็นโปรเจกต์เดียว**: ระบบจัดการห้องสมุด (Library Management
System) แบบ OOP เต็มรูปแบบ

**ต่อไป:** [Part 55 — โปรเจกต์ Library Management System](./part-055-library-system-project.md)
