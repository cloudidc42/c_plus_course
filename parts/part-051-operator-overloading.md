# Part 51: Operator Overloading (Step 401–408)

> Module D — เริ่มต้น C++ และ OOP | Part 51 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 401–408
> Part ก่อนหน้า: [Part 50 — Abstract Class และ Interface](./part-050-abstract-classes.md) | Part ถัดไป: [Part 52 — Friend Function และ Friend Class](./part-052-friend-functions.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายหลักการ "overload operator ให้สมเหตุสมผล" และรู้จักตัวอย่างของการ overload
   ที่ขัดสามัญสำนึก (Principle of Least Astonishment)
2. Overload `operator+`, `operator-`, `operator==`, `operator!=` ให้กับ class ของตัวเองได้
3. Overload `operator<<` เพื่อให้ใช้กับ `std::cout` ได้อย่างเป็นธรรมชาติ พร้อมเข้าใจว่าทำไม
   มันต้องเป็น non-member function เสมอ
4. Overload `operator[]` ทั้งเวอร์ชัน const และ non-const ได้อย่างถูกต้อง
5. Overload `operator++` ทั้งแบบ prefix และ postfix พร้อมเข้าใจ syntax ที่ใช้แยกความต่าง
6. ตัดสินใจได้ว่า operator แต่ละตัวควรเขียนเป็น **member function** หรือ **non-member
   (friend) function** และรู้เหตุผลเบื้องหลัง
7. เข้าใจแนวคิดเริ่มต้นของ **Rule of Three**: ถ้า class มี destructor หรือ copy constructor
   ที่ custom เขียนเอง ต้องคิดถึง copy assignment operator ด้วยเสมอ

---

## 51.1 หลักการ Overload Operator ให้สมเหตุสมผล (Step 401)

C++ อนุญาตให้เรา "สอน" operator มาตรฐานของภาษา (`+`, `-`, `==`, `<<`, `[]`, `++` ฯลฯ)
ให้ทำงานกับ class ที่เราสร้างเองได้ ความสามารถนี้เรียกว่า **Operator Overloading** มันทำให้
โค้ดของเราอ่านเป็นธรรมชาติมากขึ้นมาก เช่น การเขียน `a + b` แทนที่จะต้องเขียน
`a.add(b)` หรือ `add(a, b)`

แต่ความสามารถนี้มาพร้อมกับความรับผิดชอบ: **operator ที่ overload แล้วต้องทำในสิ่งที่คนอ่าน
โค้ดคาดหวังจากสัญลักษณ์นั้นเสมอ** หลักการนี้เรียกว่า **Principle of Least Astonishment**
(หลักการความประหลาดใจน้อยที่สุด) — เมื่อเห็น `+` คนอ่านโค้ดควรคาดหวังว่ามันคือ "การรวมกัน"
หรือ "การบวก" ไม่ใช่การทำสิ่งที่ไม่เกี่ยวข้องกันเลย

มาดูตัวอย่างของการ overload ที่ **ผิดหลักการอย่างรุนแรง**:

```cpp
// ตัวอย่าง ANTI-PATTERN: overload operator ให้ทำสิ่งที่ขัดสามัญสำนึก
// (คอมไพล์ผ่านและรันได้ แต่เป็นตัวอย่างของ "สิ่งที่ไม่ควรทำ")
#include <iostream>

class ShoppingCart {
public:
    explicit ShoppingCart(int item_count = 0) : item_count_(item_count) {}

    // แย่มาก: operator+ ทำให้ตะกร้าว่างเปล่า (ขัดกับความคาดหวังของทุกคนที่เห็น +)
    // คนอ่านโค้ดคาดหวังว่า cart1 + cart2 จะรวมของสองตะกร้าเข้าด้วยกัน
    // แต่โค้ดนี้กลับ "เคลียร์ตะกร้า" ซึ่งเป็นพฤติกรรมที่คาดเดาไม่ได้เลย
    ShoppingCart operator+(const ShoppingCart& /* rhs */) const {
        return ShoppingCart(0);   // ทำสิ่งที่ไม่มีใครคาดคิด!
    }

    int item_count() const { return item_count_; }

private:
    int item_count_;
};

int main() {
    ShoppingCart cart1(5);
    ShoppingCart cart2(3);

    // ผู้อ่านโค้ดคนใหม่จะคิดว่านี่คือการรวมสินค้า 5 + 3 = 8 ชิ้น
    ShoppingCart merged = cart1 + cart2;

    std::cout << "จำนวนสินค้าหลังบวก: " << merged.item_count()
              << " (ผิดความคาดหมายอย่างรุนแรง!)\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 bad_overload.cpp -o bad_overload
./bad_overload
# จำนวนสินค้าหลังบวก: 0 (ผิดความคาดหมายอย่างรุนแรง!)
```

โค้ดนี้ **compile ผ่านและรันได้ปกติ** — คอมไพเลอร์ไม่มีทางรู้เลยว่า `operator+` "ควร" ทำ
อะไร มันแค่เรียกใช้ function ที่เราเขียนไว้ ปัญหาทั้งหมดจึงตกอยู่ที่ **ความรับผิดชอบของ
โปรแกรมเมอร์** ที่ต้องออกแบบให้สมเหตุสมผล

### กฎการออกแบบที่ควรทำตามเสมอ

1. **overload เฉพาะเมื่อความหมายชัดเจนและเป็นธรรมชาติ** ถ้าไม่มีความหมายที่ชัดเจนสำหรับ
   operator นั้นกับ class ของคุณ อย่า overload มันเลย ใช้ named function เช่น
   `merge()`, `combine()` แทนจะสื่อความหมายชัดกว่ามาก
2. **รักษาความสัมพันธ์ระหว่าง operator ที่เกี่ยวข้องกัน** ถ้า overload `operator==` ต้อง
   overload `operator!=` ให้สอดคล้องกันด้วย (ปกติเขียน `!=` เป็น `!(a == b)`)
3. **รักษา "เอกลักษณ์" ทางคณิตศาสตร์ที่คนคุ้นเคย** เช่น `a + b` ควรให้ผลเหมือน `b + a`
   ถ้า operator นั้นเป็น commutative ตามธรรมชาติของสิ่งที่มันแทน (ตัวเลข, เวกเตอร์)
4. **`operator+` ไม่ควรแก้ไข operand เดิม** ควรคืนค่าใหม่กลับมาเสมอ (เหมือน `int a = 1;
   int b = a + 1;` ที่ `a` ไม่เปลี่ยนค่า) ส่วน `operator+=` ต่างหากที่ควรแก้ไข object เดิม
5. **`operator<<` สำหรับพิมพ์ค่า ไม่ควรมี side effect อื่นนอกจากพิมพ์** เช่นไม่ควรไปแก้ไขค่า
   ภายใน object ระหว่างพิมพ์

---

## 51.2 Overload `operator+` และ `operator-` ด้วย Vector2D (Step 402)

มาสร้าง class `Vector2D` ที่แทนเวกเตอร์ 2 มิติ พร้อม overload `operator+`, `operator-`
และ `operator+=` ที่มีความหมายตรงตามสามัญสำนึกทางคณิตศาสตร์ทุกประการ:

```cpp
#include <iostream>
#include <cmath>

class Vector2D {
public:
    Vector2D(double x = 0.0, double y = 0.0) : x_(x), y_(y) {}

    double x() const { return x_; }
    double y() const { return y_; }

    // member function overload: เหมาะกับ operator ที่ operand ซ้ายเป็น class เดียวกันเสมอ
    Vector2D operator+(const Vector2D& rhs) const {
        return Vector2D(x_ + rhs.x_, y_ + rhs.y_);
    }

    Vector2D operator-(const Vector2D& rhs) const {
        return Vector2D(x_ - rhs.x_, y_ - rhs.y_);
    }

    // compound assignment: แก้ไข object ตัวเอง แล้ว return reference เพื่อ chain ได้
    Vector2D& operator+=(const Vector2D& rhs) {
        x_ += rhs.x_;
        y_ += rhs.y_;
        return *this;
    }

    double length() const {
        return std::sqrt(x_ * x_ + y_ * y_);
    }

private:
    double x_;
    double y_;
};
```

สังเกตรายละเอียดสำคัญ 3 จุด:

- `operator+` และ `operator-` เป็น **`const` member function** — ไม่แก้ไข object ตัวเอง
  แต่สร้าง `Vector2D` ตัวใหม่คืนกลับไป (ตรงกับความคาดหวังของคำว่า "บวก"/"ลบ" ในทางคณิตศาสตร์)
- `operator+=` แก้ไข `x_`, `y_` ของ object ตัวเองจริงๆ แล้ว `return *this;` เพื่อให้ chain
  การเรียกต่อกันได้ เช่น `a += b += c;` (แม้ในทางปฏิบัติจะไม่ค่อยเขียนแบบนี้ก็ตาม)
- การ return `Vector2D&` (reference) จาก `operator+=` เทียบเท่ากับพฤติกรรมของ `int`
  ที่ `a += b` คืนค่า `a` หลังบวกกลับมาเช่นกัน — การ overload ที่ดีควร "เลียนแบบ" พฤติกรรม
  ของ operator ตัวเดียวกันที่ใช้กับ built-in type เสมอ

---

## 51.3 Overload `operator==` และ `operator!=` (Step 403)

การเปรียบเทียบความเท่ากันของ object สอง object ต้องอาศัย `operator==` ที่เราเขียนเอง เพราะ
คอมไพเลอร์ไม่รู้ว่า "เท่ากัน" สำหรับ class ของเราแปลว่าอะไร (สำหรับ `Vector2D` แปลว่า
พิกัด x และ y เท่ากันทั้งคู่):

```cpp
bool operator==(const Vector2D& rhs) const {
    return x_ == rhs.x_ && y_ == rhs.y_;
}

bool operator!=(const Vector2D& rhs) const {
    return !(*this == rhs);   // เขียน != จาก == เสมอ เพื่อไม่ให้ logic ทั้งสองขัดแย้งกัน
}
```

> **หมายเหตุสำหรับ C++20**: ตั้งแต่ C++20 เป็นต้นไป คอมไพเลอร์สามารถสร้าง `operator!=`
> ให้อัตโนมัติจาก `operator==` ได้เอง (rewritten candidate) ทำให้ไม่ต้องเขียน `operator!=`
> เองอีกต่อไป แต่หลักสูตรนี้ยึดมาตรฐาน C++17 เป็นหลัก ซึ่งยังต้องเขียนทั้งคู่เอง — และการ
> เขียนเองแบบนี้ก็ยังใช้ได้ปกติแม้ใน C++20 ขึ้นไป จึงเป็นแนวทางที่ปลอดภัยที่สุดเมื่อยังไม่
> แน่ใจว่าโปรเจกต์จะ compile ด้วยมาตรฐานไหนในอนาคต

การเปรียบเทียบ `double` ด้วย `==` ตรงๆ แบบนี้ (ไม่ใช้ epsilon tolerance) เหมาะกับตัวอย่าง
ในบทเรียนที่ค่าที่ป้อนเข้ามาเป็นค่าคงที่ชัดเจน แต่ในงานจริงที่ค่า `double` มาจากการคำนวณ
หลายขั้นตอน ควรเทียบด้วยค่าความคลาดเคลื่อนที่ยอมรับได้ (epsilon) แทน ซึ่งจะกลับมาพูดถึง
อย่างละเอียดใน Module เรื่อง Floating-Point Precision ต่อไป

---

## 51.4 Overload `operator<<` สำหรับ `std::cout` (Step 404)

การพิมพ์ object ของเราผ่าน `std::cout << obj` เป็นสิ่งที่ทุกคนอยากทำได้ตั้งแต่เริ่มเขียน class
แต่มีรายละเอียดสำคัญที่ต้องเข้าใจก่อน: **`operator<<` ต้องเขียนเป็น non-member function
เสมอ ไม่สามารถเขียนเป็น member function ของ `Vector2D` ได้**

เหตุผลคือไวยากรณ์ของ `std::cout << v` จริงๆ แล้วเทียบเท่ากับ `std::cout.operator<<(v)`
ถ้าจะเขียนเป็น member function มันต้องเป็น member ของ `std::ostream` (ซึ่งเป็น class ของ
Standard Library ที่เราแก้ไขไม่ได้) ไม่ใช่ member ของ `Vector2D`

```cpp
// non-member (friend) overload: จำเป็นเพราะ operand ซ้ายคือ std::ostream ไม่ใช่ Vector2D
std::ostream& operator<<(std::ostream& os, const Vector2D& v) {
    os << "(" << v.x() << ", " << v.y() << ")";
    return os;
}
```

ฟังก์ชันนี้เขียนอยู่ **นอก** class `Vector2D` และใช้แค่ `v.x()`/`v.y()` ที่เป็น public getter
เท่านั้น (ไม่จำเป็นต้องเป็น `friend` เพราะไม่ได้เข้าถึง private data ตรงๆ — เราจะเจาะลึกเรื่อง
`friend` อย่างเป็นทางการใน Part 52) การ `return os;` ท้ายฟังก์ชันสำคัญมาก เพราะทำให้
สามารถ chain การพิมพ์หลายค่าต่อกันได้ เช่น `std::cout << a << " และ " << b << '\n';`

มารวมทุกอย่างเข้าด้วยกันเป็นโปรแกรมที่สมบูรณ์:

```cpp
#include <iostream>
#include <cmath>

class Vector2D {
public:
    Vector2D(double x = 0.0, double y = 0.0) : x_(x), y_(y) {}

    double x() const { return x_; }
    double y() const { return y_; }

    Vector2D operator+(const Vector2D& rhs) const {
        return Vector2D(x_ + rhs.x_, y_ + rhs.y_);
    }

    Vector2D operator-(const Vector2D& rhs) const {
        return Vector2D(x_ - rhs.x_, y_ - rhs.y_);
    }

    Vector2D& operator+=(const Vector2D& rhs) {
        x_ += rhs.x_;
        y_ += rhs.y_;
        return *this;
    }

    bool operator==(const Vector2D& rhs) const {
        return x_ == rhs.x_ && y_ == rhs.y_;
    }

    bool operator!=(const Vector2D& rhs) const {
        return !(*this == rhs);
    }

    double length() const {
        return std::sqrt(x_ * x_ + y_ * y_);
    }

private:
    double x_;
    double y_;
};

std::ostream& operator<<(std::ostream& os, const Vector2D& v) {
    os << "(" << v.x() << ", " << v.y() << ")";
    return os;
}

int main() {
    Vector2D a(1.0, 2.0);
    Vector2D b(3.0, 4.0);

    Vector2D sum = a + b;
    Vector2D diff = b - a;

    std::cout << "a = " << a << '\n';
    std::cout << "b = " << b << '\n';
    std::cout << "a + b = " << sum << '\n';
    std::cout << "b - a = " << diff << '\n';
    std::cout << "|b| = " << b.length() << '\n';

    a += b;
    std::cout << "a += b -> a = " << a << '\n';

    std::cout << std::boolalpha;
    std::cout << "a == b ? " << (a == b) << '\n';
    std::cout << "a != b ? " << (a != b) << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 vector2d.cpp -o vector2d
./vector2d
```

ผลลัพธ์:

```
a = (1, 2)
b = (3, 4)
a + b = (4, 6)
b - a = (2, 2)
|b| = 5
a += b -> a = (4, 6)
a == b ? false
a != b ? true
```

---

## 51.5 Member vs Non-member Overload: เมื่อไหร่ต้องใช้แบบไหน (Step 405)

จากตัวอย่างที่ผ่านมา เราเห็นทั้ง operator ที่เขียนเป็น **member function** (`operator+`,
`operator==`) และที่ต้องเขียนเป็น **non-member function** (`operator<<`) กฎการตัดสินใจ
หลักมีอยู่ 2 ข้อ:

| กรณี | ควรเขียนแบบไหน | เหตุผล |
|---|---|---|
| Operand ซ้าย (`lhs`) เป็น object ของ class เราเองเสมอ | Member function | `a + b` เทียบเท่า `a.operator+(b)` ธรรมชาติอยู่แล้ว |
| Operand ซ้าย **ไม่ใช่** object ของ class เรา (เช่น `std::ostream`, หรือ `int` ที่อยู่ซ้าย) | Non-member function | ไม่มีสิทธิ์เพิ่ม member function ให้ class ที่เราไม่ได้เขียน |
| ต้องการให้ operator รองรับการแปลงชนิดข้อมูล (implicit conversion) ทั้งสองฝั่ง เช่น `5 + vec` และ `vec + 5` | Non-member function | member function รองรับได้แค่ `vec + 5` (ซ้ายต้องเป็น `Vector2D` เสมอ) |
| Operator ต้องเข้าถึง private data ของ class แต่ operand ซ้ายไม่ใช่ class นั้น | Non-member function + `friend` | จะเจาะลึกเรื่องนี้เต็มรูปแบบใน Part 52 |
| Operator ที่เปลี่ยน "state" ของ object ตัวเอง เช่น `operator+=`, `operator++`, `operator[]` | Member function เกือบเสมอ | เพราะ operand ซ้ายต้องเป็น object ของเราที่กำลังถูกแก้ไขอยู่แล้ว |

ตัวอย่างที่ชัดเจนของ "การรองรับการแปลงชนิดข้อมูลทั้งสองฝั่ง": สมมติเราต้องการให้
`Vector2D` คูณกับตัวเลข (`double`) ได้ทั้ง `v * 2.0` และ `2.0 * v`

```cpp
class Vector2D {
public:
    // ... (เหมือนเดิม)

    // รองรับแค่ v * 2.0 (operand ซ้ายเป็น Vector2D เท่านั้น)
    Vector2D operator*(double scalar) const {
        return Vector2D(x_ * scalar, y_ * scalar);
    }
};

// ต้องเขียนแยกเป็น non-member เพื่อรองรับ 2.0 * v (operand ซ้ายเป็น double)
Vector2D operator*(double scalar, const Vector2D& v) {
    return v * scalar;   // เรียก member version ที่มีอยู่แล้วซ้ำ (ไม่ต้อง implement ซ้ำ)
}
```

ถ้าเขียน `operator*` เป็น member function เพียงอย่างเดียว นิพจน์ `v * 2.0` จะ compile ผ่าน
(เพราะ compiler แปลง `v * 2.0` เป็น `v.operator*(2.0)` ได้) แต่ `2.0 * v` จะ compile **ไม่ผ่าน**
เพราะ `double` ไม่มี member function `operator*` ที่รับ `Vector2D` เป็น parameter — นี่คือ
เหตุผลที่ต้องมี non-member function อีกตัวสำหรับรองรับกรณีที่ operand ซ้ายไม่ใช่ class ของเรา

---

## 51.6 Overload `operator[]` (Step 406)

`operator[]` ใช้สำหรับให้ class ของเรา "ทำตัวเหมือน array" ได้ เช่นเดียวกับที่
`std::vector` ทำ สิ่งสำคัญคือควร overload **สองเวอร์ชัน**: เวอร์ชันที่แก้ไขค่าได้
(non-const) และเวอร์ชันที่อ่านค่าอย่างเดียว (const)

```cpp
#include <iostream>
#include <stdexcept>

class IntArray {
public:
    explicit IntArray(std::size_t size) : size_(size), data_(new int[size]{}) {}

    ~IntArray() {
        delete[] data_;
    }

    int& operator[](std::size_t index) {
        if (index >= size_) {
            throw std::out_of_range("IntArray index out of range");
        }
        return data_[index];    // return เป็น reference: อนุญาตให้แก้ไขค่าได้ เช่น arr[0] = 5;
    }

    int operator[](std::size_t index) const {
        if (index >= size_) {
            throw std::out_of_range("IntArray index out of range");
        }
        return data_[index];    // return เป็น value: object แบบ const อ่านค่าได้อย่างเดียว
    }

    std::size_t size() const { return size_; }

private:
    std::size_t size_;
    int* data_;
};
```

การมี `operator[]` สองเวอร์ชันแบบนี้เป็น pattern มาตรฐานของ Standard Library เอง
(`std::vector::operator[]` ก็ทำแบบเดียวกัน) — เวอร์ชัน non-const คืนค่าเป็น `int&` เพื่อให้
เขียน `arr[0] = 100;` ได้ ในขณะที่เวอร์ชัน const คืนค่าเป็น `int` ธรรมดา (ไม่ใช่ reference)
เพื่อป้องกันไม่ให้มีใครแก้ไขข้อมูลภายในผ่าน object ที่ประกาศเป็น `const` ได้

การตรวจสอบ `index >= size_` แล้ว `throw std::out_of_range` เป็นการป้องกันตัวเองจาก
Undefined Behavior — ถ้าไม่ตรวจสอบเลย การเข้าถึง `data_[100]` ในขณะที่ array มีแค่ 5 ช่อง
จะเป็น Undefined Behavior ทันที (อ่าน/เขียน memory นอกขอบเขตที่จองไว้)

---

## 51.7 Overload `operator++`: Prefix vs Postfix (Step 407)

C++ มี syntax พิเศษที่ค่อนข้างแปลกสำหรับแยก prefix (`++c`) กับ postfix (`c++`) ออกจากกัน
ทั้งที่ชื่อ operator เหมือนกันทุกประการ:

```cpp
class Counter {
public:
    explicit Counter(int value = 0) : value_(value) {}

    int value() const { return value_; }

    // prefix ++c : เพิ่มค่าก่อน แล้วคืน reference ของตัวเอง (ไม่ต้อง copy)
    Counter& operator++() {
        ++value_;
        return *this;
    }

    // postfix c++ : รับ int หลอก (dummy parameter) เพื่อแยกจาก prefix
    // ต้อง copy ค่าเดิมไว้ก่อนเพิ่ม แล้วคืนค่าที่ copy ไว้ (คืนเป็น value ไม่ใช่ reference)
    Counter operator++(int) {
        Counter old = *this;
        ++value_;
        return old;
    }

private:
    int value_;
};
```

จุดสังเกตสำคัญ:

- **Prefix (`operator++()`)** ไม่มี parameter เลย และ return เป็น `Counter&` (reference)
  เพราะหลังเพิ่มค่าแล้ว มันคือ object ตัวเดิมนั่นเอง ไม่จำเป็นต้อง copy ใหม่ ทำให้ **prefix
  เร็วกว่า postfix เสมอ**
- **Postfix (`operator++(int)`)** มี parameter เป็น `int` แต่ **ไม่เคยถูกใช้งานจริงในตัว
  ฟังก์ชันเลย** — มันเป็นแค่ "เครื่องหมาย" ที่คอมไพเลอร์ใช้แยกว่านี่คือ overload แบบ postfix
  (การมี parameter ที่ไม่ตั้งชื่อแบบนี้เป็น syntax พิเศษเฉพาะของ operator นี้เท่านั้น)
- Postfix ต้อง **copy ค่าเดิมเก็บไว้ก่อน** (`Counter old = *this;`) แล้วค่อยเพิ่มค่าจริง
  จากนั้น return ค่าที่ copy ไว้ (ค่า**ก่อน**เพิ่ม) กลับไป ตรงตามความหมายทางคณิตศาสตร์ของ
  `c++` ที่ "คืนค่าเดิม แล้วค่อยเพิ่มทีหลัง"

```cpp
int main() {
    Counter c(10);
    Counter after_pre = ++c;      // c เป็น 11, after_pre เป็น 11
    Counter after_post = c++;     // c เป็น 12, after_post เป็น 11 (ค่าก่อนเพิ่ม)

    std::cout << "c = " << c.value()
              << ", after_pre = " << after_pre.value()
              << ", after_post = " << after_post.value() << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 counter.cpp -o counter
./counter
# c = 12, after_pre = 11, after_post = 11
```

> **แนวทางปฏิบัติ**: ในลูปหรือโค้ดทั่วไปที่ไม่ได้ใช้ค่าที่ return จาก `++` เลย (เช่น
> `for (int i = 0; i < n; i++)`) ควรใช้ **prefix (`++i`)** เป็นค่าเริ่มต้นเสมอ เพราะเร็วกว่า
> เล็กน้อย (ไม่ต้อง copy object เก่าทิ้ง) แม้ว่ากับ `int` ธรรมดาจะไม่มีความต่างด้านประสิทธิภาพ
> เลยก็ตาม แต่กับ class ที่ copy แพง (เช่น iterator ของ container ขนาดใหญ่) ความต่างนี้
> มีนัยสำคัญจริง

---

## 51.8 Rule of Three: เกริ่นนำ (Step 408)

ตัวอย่าง `IntArray` ใน 51.6 มีจุดที่ซ่อนปัญหาใหญ่ไว้: มันจอง memory ด้วย `new int[size]`
ใน constructor และคืนด้วย `delete[]` ใน destructor แต่เรา **ไม่ได้เขียน copy constructor
หรือ copy assignment operator เองเลย**

```cpp
IntArray a(5);
IntArray b = a;    // เรียก copy constructor ที่ compiler สร้างให้อัตโนมัติ (default)
```

Copy constructor ที่ compiler สร้างให้อัตโนมัติจะทำ **Shallow Copy** — copy ค่า pointer
`data_` ตรงๆ (copy แค่ที่อยู่ ไม่ได้จองหน่วยความจำใหม่) ผลคือ `a.data_` และ `b.data_`
จะชี้ไปที่ memory เดียวกัน! เมื่อ `a` กับ `b` ถูกทำลายพร้อมกันตอนจบ scope destructor ของ
ทั้งคู่จะพยายาม `delete[]` memory เดียวกันซ้ำสองครั้ง เกิด **Double Free** ซึ่งเป็น
Undefined Behavior ที่อันตรายมาก (โปรแกรมอาจ crash หรือทำงานผิดพลาดแบบสุ่ม)

นี่คือที่มาของกฎที่เรียกว่า **Rule of Three**:

> ถ้า class ของคุณต้องเขียน **อย่างใดอย่างหนึ่ง** ต่อไปนี้เอง (custom):
> **Destructor**, **Copy Constructor**, หรือ **Copy Assignment Operator**
> แสดงว่าเกือบแน่นอนว่าคุณต้องเขียน **อีกสองอย่างที่เหลือด้วย**

เหตุผลคือถ้า class มี destructor ที่ custom (เช่น `delete[] data_;`) แสดงว่า class นั้น
"เป็นเจ้าของทรัพยากร" บางอย่างที่ไม่ใช่แค่ค่าธรรมดา (raw pointer ที่ต้องจัดการเอง) ซึ่ง
พฤติกรรม copy แบบ default (shallow copy) จะผิดพลาดเสมอสำหรับ class แบบนี้

มาเขียน `IntArray` ให้ครบตาม Rule of Three:

```cpp
#include <iostream>
#include <stdexcept>

class IntArray {
public:
    explicit IntArray(std::size_t size) : size_(size), data_(new int[size]{}) {}

    ~IntArray() {
        delete[] data_;
    }

    // Rule of Three: มี destructor ที่ custom (delete[]) จึงต้องมี copy constructor
    // และ copy assignment ที่ custom ด้วย ไม่งั้นจะเกิด double free
    IntArray(const IntArray& other) : size_(other.size_), data_(new int[other.size_]) {
        for (std::size_t i = 0; i < size_; ++i) {
            data_[i] = other.data_[i];
        }
    }

    IntArray& operator=(const IntArray& other) {
        if (this == &other) {
            return *this;   // ป้องกัน self-assignment
        }
        int* new_data = new int[other.size_];
        for (std::size_t i = 0; i < other.size_; ++i) {
            new_data[i] = other.data_[i];
        }
        delete[] data_;
        data_ = new_data;
        size_ = other.size_;
        return *this;
    }

    int& operator[](std::size_t index) {
        if (index >= size_) {
            throw std::out_of_range("IntArray index out of range");
        }
        return data_[index];
    }

    int operator[](std::size_t index) const {
        if (index >= size_) {
            throw std::out_of_range("IntArray index out of range");
        }
        return data_[index];
    }

    std::size_t size() const { return size_; }

private:
    std::size_t size_;
    int* data_;
};

int main() {
    IntArray arr(5);
    for (std::size_t i = 0; i < arr.size(); ++i) {
        arr[i] = static_cast<int>(i * 10);
    }

    IntArray copy = arr;      // เรียก copy constructor
    copy[0] = 999;            // แก้ copy แล้ว arr ต้องไม่เปลี่ยน (เพราะเป็น Deep Copy)

    std::cout << "arr:  ";
    for (std::size_t i = 0; i < arr.size(); ++i) {
        std::cout << arr[i] << " ";
    }
    std::cout << '\n';

    std::cout << "copy: ";
    for (std::size_t i = 0; i < copy.size(); ++i) {
        std::cout << copy[i] << " ";
    }
    std::cout << '\n';

    IntArray assigned(2);
    assigned = arr;           // เรียก copy assignment
    std::cout << "assigned size = " << assigned.size() << '\n';

    try {
        std::cout << arr[100] << '\n';
    } catch (const std::out_of_range& e) {
        std::cout << "จับ exception ได้: " << e.what() << '\n';
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 intarray.cpp -o intarray
./intarray
```

ผลลัพธ์:

```
arr:  0 10 20 30 40
copy: 999 10 20 30 40
assigned size = 5
จับ exception ได้: IntArray index out of range
```

สังเกตว่า `copy[0] = 999;` ไม่กระทบ `arr[0]` เลย (ยังเป็น `0` เหมือนเดิม) เพราะตอนนี้
copy constructor ทำ **Deep Copy** จริง — จองหน่วยความจำใหม่ก้อนหนึ่งแล้ว copy ค่าทีละตัว
ไม่ใช่แค่ copy ค่า pointer

การเช็ค `if (this == &other)` ใน copy assignment operator เป็นการป้องกัน
**Self-Assignment** (กรณี `a = a;`) — ถ้าไม่เช็คไว้ โค้ดจะ `delete[] data_` (ทำลายข้อมูลของ
`other` ไปด้วยเพราะเป็น object เดียวกัน) ก่อนที่จะพยายาม copy ค่าจากมัน ทำให้เกิดข้อมูลเสีย

> **หมายเหตุสำหรับอนาคต**: ตั้งแต่ C++11 เป็นต้นไป มีแนวคิด **Rule of Five** ที่เพิ่ม
> Move Constructor และ Move Assignment Operator เข้ามาด้วย (รวมเป็น 5 อย่าง) เพื่อให้
> การย้ายข้อมูล (ไม่ใช่การ copy) มีประสิทธิภาพสูงสุด เราจะเรียนเรื่องนี้อย่างเต็มรูปแบบ
> ใน Part 70 (Move Semantics และ Rvalue Reference) — ตอนนี้ให้โฟกัสที่ Rule of Three
> ให้แน่นก่อน เพราะเป็นรากฐานที่ Rule of Five ต่อยอดขึ้นไป

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **overload `operator+` แล้วแก้ไข operand เดิมโดยไม่ตั้งใจ** — `operator+` ควรเป็น
   `const` member function เสมอและคืนค่าใหม่ ไม่ใช่แก้ไข `*this` เหมือนที่ `operator+=`
   ทำ ถ้าสับสนระหว่างสองตัวนี้ โค้ดที่เรียก `a + b` จะทำให้ `a` เปลี่ยนค่าโดยไม่มีใครคาดคิด
2. **เขียน `operator<<` เป็น member function** — จะ compile ไม่ผ่านทันที เพราะ syntax
   `std::cout << obj` ต้องการให้ `std::cout` (ซึ่งเป็น `std::ostream`) เป็นฝ่ายเรียก
   `operator<<` ไม่ใช่ `obj` การแก้ไขคือย้ายออกมาเป็น non-member function เสมอ
3. **ลืม overload `operator[]` เวอร์ชัน `const`** — ทำให้ function ที่รับ
   `const IntArray&` เป็น parameter เรียก `arr[i]` ไม่ได้เลย (compile error) เพราะ
   compiler หาเวอร์ชันที่ใช้กับ const object ไม่เจอ
4. **สับสนระหว่าง prefix กับ postfix `operator++`** — ลืมใส่ `(int)` หลอกในเวอร์ชัน
   postfix ทำให้ signature ชนกับ prefix หรือลืมว่า postfix ต้อง return by value (ไม่ใช่
   reference) เพราะ object ที่ return คือค่าที่ copy ไว้ก่อนหน้า ซึ่งกำลังจะถูกทำลายเมื่อ
   ฟังก์ชันจบ
5. **มี raw pointer เป็น member แล้วลืมเขียน copy constructor/copy assignment เอง** —
   นี่คือสาเหตุอันดับต้นๆ ของบั๊ก Double Free และ Memory Corruption ในโปรแกรม C++
   ถ้า class มี destructor ที่ทำอะไรมากกว่า `= default` ให้ตรวจสอบ Rule of Three
   ทุกครั้งโดยอัตโนมัติ
6. **ลืมเช็ค self-assignment ใน `operator=`** — กรณี `a = a;` ที่พบได้จริงในโค้ดที่ซับซ้อน
   (เช่นผ่าน reference หรือ pointer ที่ชี้ไปยัง object เดียวกันโดยไม่รู้ตัว) ถ้าไม่เช็คไว้
   อาจทำให้ข้อมูลเสียหายก่อนจะ copy จากตัวเองเสร็จ

---

## แบบฝึกหัดท้ายบท

1. เขียน class `Complex` (จำนวนเชิงซ้อน) พร้อม overload `operator+`, `operator-`,
   `operator*` (ตามสูตรการคูณจำนวนเชิงซ้อน), `operator==` และ `operator<<`
2. เพิ่ม `operator*(double scalar)` และ non-member `operator*(double, const Vector2D&)`
   ให้กับ `Vector2D` ตามตัวอย่างใน 51.5 แล้วทดสอบว่า `v * 2.0` และ `2.0 * v` ให้ผลลัพธ์
   เหมือนกัน
3. เขียน class `Fraction` (เศษส่วน) พร้อม overload `operator+` และ `operator<<` โดยให้
   ผลลัพธ์ถูกย่อเศษส่วนให้เหลือรูปที่ง่ายที่สุดเสมอ (ใช้ `std::gcd` จาก `<numeric>`)
4. อธิบายด้วยคำพูดตัวเองว่าทำไม postfix `operator++` ถึงช้ากว่า prefix `operator++`
   เสมอ แล้วยกตัวอย่างสถานการณ์ในโค้ดจริงที่ความต่างนี้มีนัยสำคัญ (คำใบ้: คิดถึง
   iterator ของ container ขนาดใหญ่)
5. เขียน class `SafeString` ที่มี raw pointer `char*` เป็น member (จำลอง string ของ
   ตัวเอง) แล้วทำตาม Rule of Three ให้ครบ (destructor, copy constructor, copy
   assignment) พร้อมทดสอบว่า copy แล้วแก้ไขตัวหนึ่งไม่กระทบอีกตัวหนึ่ง
6. หาข้อผิดพลาดในโค้ดต่อไปนี้ (มี 2 จุด) แล้วอธิบายว่าทำไมถึงเป็นปัญหา:
   ```cpp
   class Point {
   public:
       Point(int x, int y) : x_(x), y_(y) {}
       Point operator+(const Point& rhs) {   // จุดที่ 1: ขาดอะไรไป?
           x_ += rhs.x_;                      // จุดที่ 2: ทำอะไรผิด?
           y_ += rhs.y_;
           return *this;
       }
   private:
       int x_, y_;
   };
   ```

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>

class Complex {
public:
    Complex(double re = 0.0, double im = 0.0) : re_(re), im_(im) {}

    Complex operator+(const Complex& rhs) const {
        return Complex(re_ + rhs.re_, im_ + rhs.im_);
    }

    Complex operator-(const Complex& rhs) const {
        return Complex(re_ - rhs.re_, im_ - rhs.im_);
    }

    Complex operator*(const Complex& rhs) const {
        return Complex(re_ * rhs.re_ - im_ * rhs.im_,
                        re_ * rhs.im_ + im_ * rhs.re_);
    }

    bool operator==(const Complex& rhs) const {
        return re_ == rhs.re_ && im_ == rhs.im_;
    }

    double re() const { return re_; }
    double im() const { return im_; }

    friend std::ostream& operator<<(std::ostream& os, const Complex& c);

private:
    double re_;
    double im_;
};

std::ostream& operator<<(std::ostream& os, const Complex& c) {
    os << c.re_;
    if (c.im_ >= 0) {
        os << " + " << c.im_ << "i";
    } else {
        os << " - " << -c.im_ << "i";
    }
    return os;
}

int main() {
    Complex a(2.0, 3.0);
    Complex b(1.0, -4.0);

    std::cout << "a = " << a << '\n';
    std::cout << "b = " << b << '\n';
    std::cout << "a + b = " << (a + b) << '\n';
    std::cout << "a - b = " << (a - b) << '\n';
    std::cout << "a * b = " << (a * b) << '\n';

    std::cout << std::boolalpha;
    std::cout << "a == b ? " << (a == b) << '\n';
    std::cout << "a == a ? " << (a == a) << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 complex.cpp -o complex
./complex
```

ผลลัพธ์:

```
a = 2 + 3i
b = 1 - 4i
a + b = 3 - 1i
a - b = 1 + 7i
a * b = 14 - 5i
a == b ? false
a == a ? true
```

สูตรการคูณจำนวนเชิงซ้อน `(a+bi)(c+di) = (ac - bd) + (ad + bc)i` ถูกนำมาใช้ตรงๆ ใน
`operator*` ส่วน `operator<<` ถูกประกาศเป็น `friend` เพื่อเข้าถึง `c.re_`/`c.im_` ที่เป็น
private ได้โดยตรง (จะอธิบายกลไกนี้อย่างละเอียดใน Part 52)

### แนวทางเฉลยข้อ 3

```cpp
#include <iostream>
#include <numeric>

class Fraction {
public:
    Fraction(int numerator, int denominator) : num_(numerator), den_(denominator) {
        reduce();
    }

    int numerator() const { return num_; }
    int denominator() const { return den_; }

private:
    void reduce() {
        int g = std::gcd(num_, den_);
        if (g != 0) {
            num_ /= g;
            den_ /= g;
        }
    }

    int num_;
    int den_;
};

// เขียนเป็น non-member function ธรรมดา (ไม่ใช่ friend) เพราะใช้แค่ getter สาธารณะก็พอ
// นี่คือหลักการ "ควร friend เท่าที่จำเป็นเท่านั้น" ที่จะเรียนใน Part 52
Fraction operator+(const Fraction& a, const Fraction& b) {
    return Fraction(a.numerator() * b.denominator() + b.numerator() * a.denominator(),
                     a.denominator() * b.denominator());
}

std::ostream& operator<<(std::ostream& os, const Fraction& f) {
    os << f.numerator() << "/" << f.denominator();
    return os;
}

int main() {
    Fraction half(1, 2);
    Fraction third(1, 3);

    std::cout << "1/2 + 1/3 = " << (half + third) << '\n';

    Fraction reducible(2, 4);
    std::cout << "2/4 ย่อแล้ว = " << reducible << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 fraction.cpp -o fraction
./fraction
```

ผลลัพธ์:

```
1/2 + 1/3 = 5/6
2/4 ย่อแล้ว = 1/2
```

ข้อนี้จงใจเขียน `operator+` และ `operator<<` เป็น **non-member function ธรรมดา** (ไม่ใช้
`friend`) เพื่อให้เห็นว่าเมื่อ class มี public getter ที่เพียงพออยู่แล้ว (`numerator()`,
`denominator()`) เราไม่จำเป็นต้องใช้ `friend` เลย — นี่คือหลักการที่ควรยึดถือ: **ใช้ `friend`
เมื่อจำเป็นจริงๆ เท่านั้น** ซึ่งจะเป็นหัวข้อหลักของ Part ถัดไป

### เฉลยข้อ 6 (หาข้อผิดพลาด)

**จุดที่ 1**: `Point operator+(const Point& rhs)` ไม่ได้ประกาศเป็น `const` member function
ทั้งที่ `operator+` ไม่ควรแก้ไข object ตัวเองเลย (ควรเป็น `Point operator+(const Point&
rhs) const`)

**จุดที่ 2**: ภายในฟังก์ชันกลับไปแก้ไข `x_`/`y_` ของ `*this` ตรงๆ (`x_ += rhs.x_;`) ซึ่ง
เป็นพฤติกรรมของ `operator+=` ไม่ใช่ `operator+` เมื่อผู้ใช้เขียน `Point c = a + b;`
พวกเขาคาดหวังว่า `a` จะไม่เปลี่ยนแปลงเลย แต่โค้ดนี้กลับไปแก้ไข `a` ให้กลายเป็นผลบวกเสีย
เอง ขัดกับ Principle of Least Astonishment ที่อธิบายไว้ใน 51.1 อย่างชัดเจน

เวอร์ชันที่ถูกต้อง:

```cpp
Point operator+(const Point& rhs) const {
    return Point(x_ + rhs.x_, y_ + rhs.y_);   // สร้าง object ใหม่ ไม่แก้ไข *this
}
```

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจหลักการ Principle of Least Astonishment: overload operator ต้องทำในสิ่งที่คนอ่าน
  โค้ดคาดหวังจากสัญลักษณ์นั้นเสมอ พร้อมเห็นตัวอย่างของการ overload ที่ผิดหลักการอย่าง
  ชัดเจน
- Overload `operator+`, `operator-`, `operator+=`, `operator==`, `operator!=` ด้วย
  class `Vector2D` ที่สมบูรณ์
- Overload `operator<<` และเข้าใจเหตุผลว่าทำไมมันต้องเป็น non-member function เสมอ
- รู้จักกฎการตัดสินใจระหว่าง member function กับ non-member function สำหรับ operator
  แต่ละประเภท พร้อมตัวอย่างการรองรับ implicit conversion ทั้งสองฝั่ง
- Overload `operator[]` ทั้งเวอร์ชัน const และ non-const เพื่อให้ class ทำตัวเหมือน
  array ได้อย่างปลอดภัย
- Overload `operator++` ทั้ง prefix และ postfix พร้อมเข้าใจ syntax พิเศษที่ใช้แยกทั้งสอง
- เข้าใจ Rule of Three: destructor, copy constructor, copy assignment operator ต้อง
  มาด้วยกันเสมอเมื่อ class เป็นเจ้าของทรัพยากรที่ต้องจัดการเอง พร้อมเห็นตัวอย่าง Double
  Free ที่เกิดจากการละเลยกฎนี้

ใน **Part 52** เราจะเจาะลึกเรื่อง **Friend Function และ Friend Class** อย่างเป็นทางการ —
ทำไมบางครั้งฟังก์ชันภายนอก (เหมือน `operator<<` ที่เพิ่งเรียนไป) ต้องขอสิทธิ์เข้าถึง
private data ของ class โดยตรง และข้อถกเถียงว่า `friend` ทำลาย Encapsulation หรือไม่

**ต่อไป:** [Part 52 — Friend Function และ Friend Class](./part-052-friend-functions.md)
