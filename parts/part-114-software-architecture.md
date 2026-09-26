# Part 114: Software Architecture สำหรับระบบ C++ ขนาดใหญ่ (Step 905–912)

> Module J — Professional และ World-Class Practices | Part 114 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 905–912
> Part ก่อนหน้า: [Part 113 — Clean Code และ Code Review Practice สำหรับ C/C++](./part-113-clean-code-review.md) | Part ถัดไป: [Part 115 — Security ใน C/C++: ช่องโหว่ที่พบบ่อยและ Secure Coding Practice](./part-115-security-cpp.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายและยกตัวอย่างโค้ด C++ จริงของ **SOLID Principles** ทั้ง 5 ข้อ ได้แก่ Single
   Responsibility, Open-Closed, Liskov Substitution, Interface Segregation, และ Dependency
   Inversion
2. เชื่อมโยง **Open-Closed Principle** และ **Dependency Inversion Principle** เข้ากับ Abstract
   Class และ Interface ที่เรียนไปแล้วใน **Part 50** ได้อย่างเป็นรูปธรรม
3. ระบุการละเมิด **Liskov Substitution Principle** ในโค้ดที่ดู "ถูกต้องตามไวยากรณ์" แต่มีปัญหา
   เชิงพฤติกรรมซ่อนอยู่ พร้อมพิสูจน์ด้วยโค้ดที่รันแล้ว assertion ล้มเหลวจริง
4. ออกแบบ **Layered Architecture** (Presentation / Business Logic / Data Access) และนำไป
   ปรับใช้กับโปรเจกต์ REST API จาก **Part 108** เพื่อแยกกฎทางธุรกิจออกจากโค้ดที่ผูกกับ HTTP
   โดยตรง
5. implement **Dependency Injection** แบบ Constructor Injection ด้วยมือใน C++ (โดยไม่มี
   Framework ช่วยเหมือนภาษาอื่น) และอธิบายได้ว่าทำไมเทคนิคนี้ช่วยให้ทดสอบโค้ดได้ง่ายขึ้นมาก
6. อธิบายแนวคิด **Technical Debt** ได้อย่างถูกต้อง ทั้งประเภทของหนี้ทางเทคนิคและผลกระทบระยะยาว
   ต่อความเร็วในการพัฒนาซอฟต์แวร์
7. ใช้กรอบการตัดสินใจที่เป็นระบบ เพื่อเลือกว่าเมื่อไหร่ควร **Refactor** และเมื่อไหร่ควร
   **Rewrite ใหม่ทั้งหมด** แทนที่จะตัดสินใจตามความรู้สึกอย่างเดียว

> **หมายเหตุเรื่องสภาพแวดล้อมของ Part นี้**: ทุกตัวอย่างโค้ด C++ ใน Part นี้ถูกคอมไพล์และรันจริง
> บนเครื่องนี้ด้วย `g++ 13.3.0 -Wall -Wextra -Wpedantic -std=c++17` โดยไม่มี warning เหลืออยู่เลย
> (ยกเว้นตัวอย่างที่ตั้งใจสาธิตการละเมิดหลักการ ซึ่งกำกับไว้ชัดเจนว่า "ตัวอย่างที่ไม่ควรทำ") รวมถึง
> ตัวอย่างในหัวข้อ Liskov Substitution ที่จงใจให้โปรแกรม `assert` ล้มเหลวจริงเพื่อพิสูจน์ปัญหาให้เห็น
> เป็นรูปธรรม ไม่ใช่แค่คำอธิบายลอยๆ

---

## 114.1 SOLID Principles: ภาพรวมและ Single Responsibility Principle (Step 905)

### SOLID คืออะไร

**SOLID** เป็นตัวย่อของหลักการออกแบบซอฟต์แวร์เชิงวัตถุ (Object-Oriented Design) 5 ข้อ ที่ Robert
C. Martin รวบรวมไว้ตั้งแต่ต้นยุค 2000 และกลายเป็นมาตรฐานที่วิศวกร OOP ทั่วโลกยึดถือมาจนถึงปัจจุบัน
เป้าหมายร่วมของทั้ง 5 ข้อคือทำให้ระบบซอฟต์แวร์ **ยืดหยุ่นต่อการเปลี่ยนแปลง** โดยไม่ต้องแก้ไขโค้ด
เดิมที่ทำงานถูกต้องอยู่แล้วทุกครั้งที่มีความต้องการใหม่เข้ามา

| ตัวอักษร | หลักการ | สรุปใจความสำคัญ |
|---|---|---|
| **S** | Single Responsibility Principle (SRP) | คลาสหนึ่งควรมีเหตุผลเดียวที่ต้องถูกแก้ไข |
| **O** | Open-Closed Principle (OCP) | เปิดต่อการขยาย (extension) แต่ปิดต่อการแก้ไข (modification) |
| **L** | Liskov Substitution Principle (LSP) | Subclass ต้องแทนที่ Base class ได้โดยไม่ทำให้พฤติกรรมของโปรแกรมเปลี่ยนไป |
| **I** | Interface Segregation Principle (ISP) | ไม่ควรบังคับให้ class ต้อง implement เมธอดที่ตัวเองไม่ได้ใช้ |
| **D** | Dependency Inversion Principle (DIP) | โมดูลระดับสูงและโมดูลระดับล่างควรพึ่งพา "นามธรรม" ร่วมกัน ไม่ใช่พึ่งพากันโดยตรง |

หลักการเหล่านี้**ไม่ใช่กฎที่ต้องท่องจำแล้วยึดติดแบบไม่ยืดหยุ่น** (เหมือนที่เตือนไว้แล้วใน Part 80
เรื่อง C++ Core Guidelines) แต่เป็นเครื่องมือช่วยคิดที่ทำให้เราออกแบบระบบที่ **ทนต่อการเปลี่ยนแปลง
ความต้องการในอนาคต** ได้ดีขึ้น — ทุกโปรเจกต์ซอฟต์แวร์ล้วนมีความต้องการที่เปลี่ยนแปลงตลอดเวลา
SOLID คือชุดหลักการที่ช่วยให้การเปลี่ยนแปลงนั้นมีต้นทุนต่ำที่สุดเท่าที่จะเป็นไปได้

### Single Responsibility Principle (SRP): นิยามที่ถูกต้อง

หลาย ๆ คนเข้าใจ SRP ผิดว่าหมายถึง "คลาสควรทำแค่ 1 เมธอด" ซึ่ง**ไม่ถูกต้อง** นิยามที่แท้จริงของ
Robert C. Martin คือ:

> **"A class should have only one reason to change."**
> (คลาสหนึ่งควรมีเหตุผลเดียวเท่านั้นที่ทำให้ต้องถูกแก้ไข)

คลาสสามารถมีหลายเมธอดได้ตราบใดที่ทุกเมธอดนั้น**รับใช้ความรับผิดชอบเดียวกัน** สิ่งที่ต้องระวังคือ
เมื่อคลาสหนึ่งมีเหตุผลให้ต้องแก้ไข **จากหลายแหล่งที่มาที่ไม่เกี่ยวข้องกัน** (เช่น ทีมบัญชีขอให้เปลี่ยน
สูตรคำนวณ, ทีมการตลาดขอให้เปลี่ยนรูปแบบข้อความ, ทีม infrastructure ขอให้เปลี่ยนปลายทางที่เก็บไฟล์)
นั่นคือสัญญาณชัดเจนว่าคลาสนั้นละเมิด SRP

### ตัวอย่าง: คลาสที่ละเมิด SRP

```cpp
// srp_bad.cpp — ตัวอย่างที่ไม่ควรทำ
#include <fstream>
#include <iostream>
#include <string>
#include <vector>

// class เดียวรับผิดชอบ 3 เรื่องพร้อมกัน (คำนวณ, format ข้อความ, และเขียนไฟล์)
class SalesReport {
public:
    explicit SalesReport(std::vector<double> daily_sales) : daily_sales_(std::move(daily_sales)) {}

    double total() const {
        double sum = 0.0;
        for (double s : daily_sales_) {
            sum += s;
        }
        return sum;
    }

    std::string to_text() const {
        return "ยอดขายรวม: " + std::to_string(total()) + " บาท";
    }

    void save_to_file(const std::string& path) const {
        std::ofstream out(path);
        out << to_text() << "\n";
    }

private:
    std::vector<double> daily_sales_;
};

int main() {
    SalesReport report({100.0, 200.0, 150.5});
    std::cout << report.to_text() << "\n";
    report.save_to_file("/tmp/report_scratch_test.txt");
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 srp_bad.cpp -o srp_bad && ./srp_bad
```

```
ยอดขายรวม: 450.500000 บาท
```

ลองนึกถึงเหตุผล 3 อย่างที่ทำให้ต้องแก้ `SalesReport` ในอนาคต ซึ่ง**ไม่เกี่ยวข้องกันเลย**:

1. ทีมบัญชีเปลี่ยนสูตรคำนวณยอดขาย (เช่น ต้องหักภาษี ณ ที่จ่ายก่อนรวม) — ต้องแก้ `total()`
2. ทีมการตลาดต้องการรูปแบบข้อความใหม่ (เช่น ต้องมีวันที่กำกับ) — ต้องแก้ `to_text()`
3. ทีม Infrastructure ตัดสินใจย้ายจากการเขียนไฟล์ในเครื่องไปเป็นการอัปโหลดขึ้น S3 — ต้องแก้
   `save_to_file()`

ทั้ง 3 เหตุผลนี้มาจาก**คนละแหล่งที่มา**และ**คนละทีม** แต่ทั้งหมดบังคับให้ต้องแก้ไขคลาสเดียวกัน —
นี่คือสัญญาณชัดเจนของการละเมิด SRP ปัญหาที่ตามมาคือทุกครั้งที่ทีมใดทีมหนึ่งแก้โค้ดในคลาสนี้
มีความเสี่ยงที่จะทำให้ฟังก์ชันการทำงานของอีก 2 ส่วนพังโดยไม่ตั้งใจ

### เวอร์ชันที่ถูกต้อง: แยกความรับผิดชอบ

```cpp
// srp_good.cpp
#include <fstream>
#include <iomanip>
#include <iostream>
#include <sstream>
#include <string>
#include <vector>

// SRP: แยกความรับผิดชอบเป็น 3 class อิสระ แต่ละตัวมีเหตุผลเดียวที่จะต้องแก้ไข

// 1) SalesCalculator: รับผิดชอบแค่ "คำนวณ" — เปลี่ยนสูตรคำนวณ แก้ที่นี่ที่เดียว
class SalesCalculator {
public:
    explicit SalesCalculator(std::vector<double> daily_sales)
        : daily_sales_(std::move(daily_sales)) {}

    double total() const {
        double sum = 0.0;
        for (double s : daily_sales_) {
            sum += s;
        }
        return sum;
    }

private:
    std::vector<double> daily_sales_;
};

// 2) SalesReportFormatter: รับผิดชอบแค่ "จัดรูปแบบข้อความ" — เปลี่ยนภาษา/รูปแบบ แก้ที่นี่
class SalesReportFormatter {
public:
    static std::string format(double total) {
        std::ostringstream oss;
        oss << "ยอดขายรวม: " << std::fixed << std::setprecision(2) << total << " บาท";
        return oss.str();
    }
};

// 3) ReportFileWriter: รับผิดชอบแค่ "เขียนไฟล์" — เปลี่ยนปลายทาง (S3, database) แก้ที่นี่
class ReportFileWriter {
public:
    static void write(const std::string& path, const std::string& content) {
        std::ofstream out(path);
        out << content << "\n";
    }
};

int main() {
    SalesCalculator calculator({100.0, 200.0, 150.5});
    const double total = calculator.total();

    const std::string text = SalesReportFormatter::format(total);
    std::cout << text << "\n";

    ReportFileWriter::write("/tmp/report_scratch_test2.txt", text);
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 srp_good.cpp -o srp_good && ./srp_good
```

```
ยอดขายรวม: 450.50 บาท
```

ตอนนี้ทีมบัญชีแก้ `SalesCalculator` ได้โดยไม่กระทบ `SalesReportFormatter` เลย ทีมการตลาดแก้
`SalesReportFormatter` ได้โดยไม่กระทบการคำนวณ และแต่ละคลาสทดสอบด้วย Unit Test (Part 93)
แยกจากกันได้อย่างอิสระสมบูรณ์

---

## 114.2 Open-Closed Principle: เชื่อมโยงกับ Abstract Class จาก Part 50 (Step 906)

### นิยาม

> **"Software entities should be open for extension, but closed for modification."**
> (ระบบซอฟต์แวร์ควรเปิดให้ขยายความสามารถได้ แต่ปิดไม่ให้ต้องแก้ไขโค้ดเดิม)

หมายความว่า เมื่อมีความต้องการใหม่เข้ามา (เช่น เพิ่มรูปทรงใหม่, เพิ่มวิธีชำระเงินใหม่) เราควร
**เพิ่มโค้ดใหม่** ได้โดย**ไม่ต้องแก้ไขโค้ดเดิมที่ทดสอบและใช้งานอยู่แล้ว** ยิ่งแก้โค้ดเดิมน้อยเท่าไหร่
ความเสี่ยงที่จะทำฟีเจอร์เดิมพังก็ยิ่งน้อยลงเท่านั้น

### ตัวอย่าง: ละเมิด OCP ด้วย `if-else` ที่ต้องแก้ทุกครั้งที่มีของใหม่

```cpp
// ocp_bad.cpp — ตัวอย่างที่ไม่ควรทำ
#include <iostream>
#include <vector>

enum class ShapeKind { kCircle, kRectangle };

struct Shape {
    ShapeKind kind;
    double radius;         // ใช้เมื่อเป็น Circle
    double width, height;  // ใช้เมื่อเป็น Rectangle
};

double calculate_total_area(const std::vector<Shape>& shapes) {
    double total = 0.0;
    for (const auto& s : shapes) {
        if (s.kind == ShapeKind::kCircle) {
            total += 3.14159265358979 * s.radius * s.radius;
        } else if (s.kind == ShapeKind::kRectangle) {
            total += s.width * s.height;
        }
        // ถ้าจะเพิ่ม Triangle ต้องมาแก้ if-else ตรงนี้อีก — ฟังก์ชันนี้ "ไม่ปิดต่อการแก้ไข"
    }
    return total;
}

int main() {
    std::vector<Shape> shapes{
        {ShapeKind::kCircle, 2.0, 0, 0},
        {ShapeKind::kRectangle, 0, 3.0, 4.0},
    };
    std::cout << calculate_total_area(shapes) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ocp_bad.cpp -o ocp_bad && ./ocp_bad
```

```
24.5664
```

ทุกครั้งที่ต้องเพิ่มรูปทรงใหม่ (Triangle, Pentagon, ...) ต้องเปิดฟังก์ชัน `calculate_total_area`
ขึ้นมาแก้ไขโดยตรง เสี่ยงทำ logic ของ `Circle`/`Rectangle` ที่ทำงานถูกต้องอยู่แล้วพังไปด้วยโดยไม่
ตั้งใจ (โดยเฉพาะถ้าไฟล์นี้มี `if-else` ยาวนับสิบ-นับร้อยบรรทัดสำหรับรูปทรงหลายสิบชนิด)

### เวอร์ชันที่ถูกต้อง: ใช้ Abstract Class จาก Part 50

```cpp
// ocp_good.cpp
#include <iostream>
#include <memory>
#include <vector>

// OCP: ใช้ abstract class (Part 50) แทน enum + if-else
// calculate_total_area() ไม่ต้องแก้ไขเลยแม้จะเพิ่มรูปทรงใหม่กี่ชนิดก็ตาม
// "ปิดต่อการแก้ไข (Closed for modification), เปิดต่อการขยาย (Open for extension)"
class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    explicit Circle(double radius) : radius_(radius) {}
    double area() const override { return 3.14159265358979 * radius_ * radius_; }

private:
    double radius_;
};

class Rectangle : public Shape {
public:
    Rectangle(double width, double height) : width_(width), height_(height) {}
    double area() const override { return width_ * height_; }

private:
    double width_;
    double height_;
};

// เพิ่มรูปทรงใหม่: แค่สร้าง class ใหม่ ไม่ต้องแตะฟังก์ชันเดิมแม้แต่บรรทัดเดียว
class Triangle : public Shape {
public:
    Triangle(double base, double height) : base_(base), height_(height) {}
    double area() const override { return 0.5 * base_ * height_; }

private:
    double base_;
    double height_;
};

double calculate_total_area(const std::vector<std::unique_ptr<Shape>>& shapes) {
    double total = 0.0;
    for (const auto& s : shapes) {
        total += s->area();  // Dynamic Dispatch — ไม่ต้องรู้ชนิดจริงของ s เลย
    }
    return total;
}

int main() {
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>(2.0));
    shapes.push_back(std::make_unique<Rectangle>(3.0, 4.0));
    shapes.push_back(std::make_unique<Triangle>(5.0, 6.0));

    std::cout << calculate_total_area(shapes) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ocp_good.cpp -o ocp_good && ./ocp_good
```

```
39.5664
```

สังเกตว่าเราเพิ่ม `Triangle` เข้าไปในระบบได้โดย `calculate_total_area` **ไม่ถูกแก้ไขแม้แต่ตัวอักษร
เดียว** เพราะฟังก์ชันนี้พึ่งพาแค่ **สัญญา** ของ `Shape` (มี `area()`) ไม่ใช่พึ่งพารายละเอียดของแต่ละ
รูปทรง — นี่คือกลไก **Dynamic Dispatch** ที่เราเรียนมาตั้งแต่ Part 49 ทำงานร่วมกับ **Abstract Class**
จาก Part 50 อย่างสมบูรณ์แบบเพื่อให้เกิด OCP ในทางปฏิบัติ

---

## 114.3 Liskov Substitution Principle: พิสูจน์ด้วยโค้ดที่พังจริง (Step 907)

### นิยาม

> **"Subtypes must be substitutable for their base types."**
> (ทุกที่ที่ใช้ Base class object ได้ ต้องสามารถแทนที่ด้วย Derived class object ได้โดยไม่ทำให้
> พฤติกรรมของโปรแกรมเปลี่ยนไปในทางที่ผิดพลาด)

LSP เป็นหลักการที่**ตรวจจับด้วยการดูไวยากรณ์เพียงอย่างเดียวไม่ได้** เพราะโค้ดที่ละเมิด LSP มักจะ
**compile ผ่านสมบูรณ์แบบและดูสมเหตุสมผลตามหลักภาษา** แต่มีปัญหาเชิง**พฤติกรรม**ซ่อนอยู่ ตัวอย่าง
คลาสสิกที่สุดในวงการ (ถูกสอนในมหาวิทยาลัยและหนังสือ OOP ทั่วโลก) คือปัญหา Rectangle/Square

### ตัวอย่าง: ละเมิด LSP ด้วย Rectangle/Square

```cpp
// lsp_bad.cpp — ตัวอย่างที่ไม่ควรทำ
#include <cassert>
#include <iostream>

// Square สืบทอดจาก Rectangle เพราะ "ในทางคณิตศาสตร์ สี่เหลี่ยมจัตุรัสก็คือสี่เหลี่ยม
// ผืนผ้าชนิดหนึ่ง" แต่การบังคับให้ width == height เสมอ ทำลาย "สัญญา" ที่ Rectangle ให้ไว้
class Rectangle {
public:
    virtual void set_width(double w) { width_ = w; }
    virtual void set_height(double h) { height_ = h; }
    double area() const { return width_ * height_; }
    virtual ~Rectangle() = default;

protected:
    double width_ = 0;
    double height_ = 0;
};

class Square : public Rectangle {
public:
    // ต้อง override เพื่อรักษากฎ "จัตุรัสด้านเท่ากันเสมอ" แต่กลับทำลาย
    // พฤติกรรมที่ผู้เรียกคาดหวังจาก Rectangle::set_width() (ที่ควรกระทบแค่ width)
    void set_width(double w) override {
        width_ = w;
        height_ = w;  // ผลข้างเคียงที่ผู้เรียกไม่คาดคิดถ้ามองผ่าน Rectangle*
    }
    void set_height(double h) override {
        width_ = h;
        height_ = h;
    }
};

// ฟังก์ชันนี้เขียนโดยอิงกับ "สัญญา" ของ Rectangle: ตั้ง width, height แยกกันได้อิสระ
void resize_and_check(Rectangle& rect) {
    rect.set_width(5.0);
    rect.set_height(4.0);
    // คาดหวัง area = 5 * 4 = 20 เสมอ ตามสัญญาของ Rectangle
    assert(rect.area() == 20.0 && "Liskov Substitution ถูกละเมิด!");
    std::cout << "area = " << rect.area() << " (ผ่านตามที่คาดหวัง)\n";
}

int main() {
    Rectangle rect;
    resize_and_check(rect);   // ผ่าน: area = 20

    Square square;
    resize_and_check(square); // พังตรงนี้: Square ไม่สามารถแทนที่ Rectangle ได้จริง
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 lsp_bad.cpp -o lsp_bad
./lsp_bad
echo "exit code: $?"
```

ผลลัพธ์จริงที่ได้เมื่อรันบนเครื่องนี้ (การ compile ผ่านสมบูรณ์ ไม่มี warning ใดๆ เลย — ปัญหานี้
compiler มองไม่เห็นเด็ดขาด เพราะเป็นปัญหาเชิงพฤติกรรม ไม่ใช่ปัญหาเชิงไวยากรณ์):

```
lsp_bad: lsp_bad.cpp:38: void resize_and_check(Rectangle&): Assertion `rect.area() == 20.0 && "Liskov Substitution ถูกละเมิด!"' failed.
Aborted (core dumped)
exit code: 134
```

นี่คือหลักฐานที่เป็นรูปธรรมที่สุด: `resize_and_check()` เขียนขึ้นโดยอ้างอิงกับ **สัญญา** ของ
`Rectangle` (ตั้ง width และ height แยกจากกันได้อิสระ แล้ว area จะเป็นผลคูณของทั้งสอง) โปรแกรม
**ทำงานถูกต้องกับ `Rectangle` จริง** (`area = 20`) แต่เมื่อแทนที่ด้วย `Square` (ซึ่งเป็น subclass
ที่ compiler ยอมรับว่า "แทนที่ได้" ตามกฎ inheritance) โปรแกรมกลับ **ล้มเหลวจริง** เพราะ `Square`
ทำลายสัญญาที่ผู้เรียกคาดหวังไว้ — นี่คือแก่นของ LSP: **การสืบทอด (inheritance) ทางไวยากรณ์ไม่ได้
รับประกันว่าจะแทนที่กันได้จริงในทางพฤติกรรม**

### เวอร์ชันที่ถูกต้อง: ไม่บังคับความสัมพันธ์แบบสืบทอดที่ไม่จริง

```cpp
// lsp_good.cpp
#include <cassert>
#include <iostream>

// แก้ตาม LSP: ไม่บังคับให้ Square "เป็น" Rectangle (is-a ที่ไม่จริง)
// ให้ทั้งคู่ implement abstract interface ร่วมกันแทน (Part 50) โดยไม่มีความสัมพันธ์
// สืบทอดกันเอง — แต่ละ class คงพฤติกรรมของตัวเองไว้ครบถ้วน ไม่มีฝ่ายใดถูกบังคับ
// ให้ทำลายสัญญาของอีกฝ่าย
class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

class Rectangle : public Shape {
public:
    Rectangle(double width, double height) : width_(width), height_(height) {}
    double area() const override { return width_ * height_; }

private:
    double width_;
    double height_;
};

class Square : public Shape {
public:
    explicit Square(double side) : side_(side) {}
    double area() const override { return side_ * side_; }

private:
    double side_;
};

// ฟังก์ชันนี้พึ่งพาแค่ "สัญญา" ของ Shape (มี area()) เท่านั้น
// ใช้ได้กับทุก Shape ที่ implement ถูกต้อง โดยไม่มีข้อยกเว้นที่ทำให้พัง
void print_area(const Shape& shape) {
    std::cout << "area = " << shape.area() << "\n";
}

int main() {
    Rectangle rect(5.0, 4.0);
    Square square(5.0);

    print_area(rect);
    print_area(square);

    assert(rect.area() == 20.0);
    assert(square.area() == 25.0);
    std::cout << "ทั้งสอง Shape ใช้งานผ่าน interface เดียวกันได้ถูกต้อง ไม่มีฝ่ายใดพัง\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 lsp_good.cpp -o lsp_good && ./lsp_good
```

```
area = 20
area = 25
ทั้งสอง Shape ใช้งานผ่าน interface เดียวกันได้ถูกต้อง ไม่มีฝ่ายใดพัง
```

### บทเรียนสำคัญจาก LSP

> **หลักการทดสอบง่ายๆ**: ถ้าลูกคลาส (derived class) ต้อง override เมธอดของแม่คลาส (base class)
> เพียงเพื่อ "ปฏิเสธ" หรือ "จำกัด" พฤติกรรมที่แม่คลาสรับปากไว้ (เช่น throw exception แทนการทำงาน
> จริง, หรือสร้าง side effect ที่แม่คลาสไม่เคยมี) นั่นคือสัญญาณเตือนว่าความสัมพันธ์แบบ **inheritance**
> อาจไม่ใช่โมเดลที่ถูกต้องสำหรับสถานการณ์นี้ — ควรพิจารณาใช้ **Composition** หรือแยกเป็น
> **Interface คนละตัว** แทน (จะเจาะลึกในหัวข้อ 114.4 ด้วย Interface Segregation Principle)

หลักการนี้ตรงกับ "Prefer Composition over Inheritance" ข้อ 18 ที่เราสรุปไว้แล้วใน **Part 80** —
Inheritance ควรใช้เมื่อมีความสัมพันธ์แบบ "is-a" ที่แท้จริงและ**แทนที่กันได้ในทางพฤติกรรม** ไม่ใช่
แค่ "ดูเหมือนจะเกี่ยวข้องกันทางความหมาย" อย่าง Rectangle กับ Square

---

## 114.4 Interface Segregation Principle (Step 908)

### นิยาม

> **"Clients should not be forced to depend upon interfaces that they do not use."**
> (ไม่ควรบังคับให้ผู้ใช้งาน (client) ต้องพึ่งพา interface ที่ตัวเองไม่ได้ใช้)

หลักการนี้เตือนไม่ให้สร้าง **Interface ขนาดใหญ่เทอะทะ** (Fat Interface) ที่รวมความสามารถหลายอย่าง
ไว้ในที่เดียว เพราะจะบังคับให้ทุก class ที่ implement ต้องมีเมธอดครบทุกตัว แม้บาง class จะทำ
ความสามารถบางอย่างไม่ได้จริงก็ตาม — นี่คือประเด็นที่ Part 50 เคยเกริ่นไว้สั้นๆ แล้วว่า "อย่าออกแบบ
interface ที่ใหญ่เทอะทะจนบังคับให้ทุก class ที่ implement ต้องมีเมธอดที่ตัวเองไม่ต้องการ" — Part นี้
จะขยายความพร้อมตัวอย่างเต็มรูปแบบ

### ตัวอย่าง: ละเมิด ISP ด้วย Interface ที่ใหญ่เกินไป

```cpp
// isp_bad.cpp — ตัวอย่างที่ไม่ควรทำ
#include <iostream>
#include <stdexcept>
#include <string>

// IMultiFunctionPrinter บังคับให้ทุก class ที่ implement ต้องมีทั้ง print, scan, fax
// ทั้งที่เครื่องพิมพ์ราคาถูกบางรุ่น "พิมพ์ได้อย่างเดียว" ไม่มี scan/fax จริง
class IMultiFunctionPrinter {
public:
    virtual void print(const std::string& doc) = 0;
    virtual void scan(const std::string& doc) = 0;
    virtual void fax(const std::string& doc) = 0;
    virtual ~IMultiFunctionPrinter() = default;
};

class BasicPrinter : public IMultiFunctionPrinter {
public:
    void print(const std::string& doc) override {
        std::cout << "พิมพ์: " << doc << "\n";
    }
    // ถูกบังคับให้ implement แม้เครื่องจริงทำไม่ได้ — เหลือแค่ throw exception
    void scan(const std::string&) override {
        throw std::logic_error("เครื่องนี้ไม่รองรับการสแกน");
    }
    void fax(const std::string&) override {
        throw std::logic_error("เครื่องนี้ไม่รองรับการส่งแฟกซ์");
    }
};

int main() {
    BasicPrinter printer;
    printer.print("รายงานประจำเดือน");
    try {
        printer.scan("รายงานประจำเดือน");
    } catch (const std::exception& e) {
        std::cout << "Error: " << e.what() << "\n";
    }
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 isp_bad.cpp -o isp_bad && ./isp_bad
```

```
พิมพ์: รายงานประจำเดือน
Error: เครื่องนี้ไม่รองรับการสแกน
```

สังเกตว่าโค้ดนี้**สอดคล้องกับสัญญาณเตือน LSP ที่เพิ่งเรียนในหัวข้อก่อน** — `BasicPrinter` ต้อง
override เมธอด `scan()`/`fax()` เพียงเพื่อ throw exception ปฏิเสธการทำงาน นี่คือรูปแบบเดียวกับ
ปัญหา Rectangle/Square: มันคือสัญญาณว่า `IMultiFunctionPrinter` **ใหญ่เกินไป** และไม่ใช่ทุก
"เครื่องพิมพ์" ที่แทนที่ interface นี้ได้จริงตามที่มันสัญญาไว้

### เวอร์ชันที่ถูกต้อง: แยก Interface ให้เล็กและเจาะจง

```cpp
// isp_good.cpp
#include <iostream>
#include <string>

// ISP: แยก interface ใหญ่ออกเป็น interface เล็กๆ ที่เฉพาะเจาะจง
// class ที่ implement เลือกได้ว่าจะ "รับปาก" (implement) อะไรบ้าง
// ไม่ถูกบังคับให้มีเมธอดที่ตัวเองทำไม่ได้จริง
class IPrinter {
public:
    virtual void print(const std::string& doc) = 0;
    virtual ~IPrinter() = default;
};

class IScanner {
public:
    virtual void scan(const std::string& doc) = 0;
    virtual ~IScanner() = default;
};

class IFax {
public:
    virtual void fax(const std::string& doc) = 0;
    virtual ~IFax() = default;
};

// เครื่องพิมพ์ราคาถูก: implement แค่ IPrinter ไม่ต้องมี scan/fax ปลอมๆ เลย
class BasicPrinter : public IPrinter {
public:
    void print(const std::string& doc) override {
        std::cout << "พิมพ์: " << doc << "\n";
    }
};

// เครื่อง All-in-One: implement ครบทั้ง 3 interface เพราะทำได้จริงทุกอย่าง
class AllInOnePrinter : public IPrinter, public IScanner, public IFax {
public:
    void print(const std::string& doc) override {
        std::cout << "พิมพ์ (All-in-One): " << doc << "\n";
    }
    void scan(const std::string& doc) override {
        std::cout << "สแกน (All-in-One): " << doc << "\n";
    }
    void fax(const std::string& doc) override {
        std::cout << "ส่งแฟกซ์ (All-in-One): " << doc << "\n";
    }
};

int main() {
    BasicPrinter basic;
    basic.print("รายงานประจำเดือน");

    AllInOnePrinter all_in_one;
    all_in_one.print("สัญญาจ้างงาน");
    all_in_one.scan("สัญญาจ้างงาน");
    all_in_one.fax("สัญญาจ้างงาน");
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 isp_good.cpp -o isp_good && ./isp_good
```

```
พิมพ์: รายงานประจำเดือน
พิมพ์ (All-in-One): สัญญาจ้างงาน
สแกน (All-in-One): สัญญาจ้างงาน
ส่งแฟกซ์ (All-in-One): สัญญาจ้างงาน
```

ตอนนี้ `BasicPrinter` implement แค่ `IPrinter` — ไม่มี exception ปลอมๆ ไม่มีเมธอดที่ทำงานไม่ได้จริง
ซ่อนอยู่เลย ในขณะที่ `AllInOnePrinter` เลือก implement ทั้ง 3 interface เพราะทำได้จริงทุกอย่าง —
**ความสามารถของแต่ละ class ถูกประกาศอย่างซื่อสัตย์ผ่านชนิดข้อมูลของมันเอง (Type System)** แทนที่
จะซ่อนข้อจำกัดไว้เป็น exception ที่รู้ได้แค่ตอน runtime เท่านั้น

---

## 114.5 Dependency Inversion Principle: เชื่อมโยงกับ Interface จาก Part 50 (Step 909)

### นิยาม

> **"High-level modules should not depend on low-level modules. Both should depend on
> abstractions."**
> (โมดูลระดับสูงไม่ควรพึ่งพาโมดูลระดับล่างโดยตรง ทั้งคู่ควรพึ่งพา "นามธรรม" ร่วมกัน)

คำว่า "Inversion" (การกลับด้าน) มาจากการที่ทิศทางการพึ่งพาถูก **กลับด้าน** จากที่เคยเป็น
`ระดับสูง → ระดับล่าง (concrete)` โดยตรง กลายเป็น `ระดับสูง → นามธรรม ← ระดับล่าง` — ทั้งสองฝั่ง
ชี้เข้าหา Interface ตรงกลาง ไม่มีฝ่ายใดรู้จักรายละเอียด implementation ของอีกฝ่ายเลย นี่คือหลักการที่
ต่อยอดโดยตรงจาก **Interface** ที่เรียนใน **Part 50** — DIP คือเหตุผลเชิงสถาปัตยกรรมที่อธิบายว่า
"ทำไมถึงควรออกแบบ Interface" ไม่ใช่แค่เพราะ "ภาษาให้ทำได้"

### ตัวอย่าง: ละเมิด DIP ด้วยการพึ่งพา Concrete Class โดยตรง

```cpp
// dip_bad.cpp — ตัวอย่างที่ไม่ควรทำ
#include <iostream>
#include <string>

// NotificationService (โมดูลระดับสูง) ผูกติดกับ EmailSender (โมดูลระดับล่าง) โดยตรง
class EmailSender {
public:
    void send(const std::string& to, const std::string& message) {
        std::cout << "[Email] ส่งถึง " << to << ": " << message << "\n";
    }
};

class NotificationService {
public:
    // สร้าง EmailSender เองภายใน — ผูกติดกับ implementation ที่เจาะจงตายตัว
    void notify(const std::string& user, const std::string& message) {
        EmailSender sender;
        sender.send(user, message);
    }
    // ถ้าอยากเปลี่ยนไปส่ง SMS แทน ต้องแก้โค้ดของ NotificationService โดยตรง
    // และไม่มีทาง unit test แยกจาก EmailSender จริงได้เลย (Part 93)
};

int main() {
    NotificationService service;
    service.notify("somchai@example.com", "ออเดอร์ของคุณจัดส่งแล้ว");
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 dip_bad.cpp -o dip_bad && ./dip_bad
```

```
[Email] ส่งถึง somchai@example.com: ออเดอร์ของคุณจัดส่งแล้ว
```

`NotificationService` (โมดูลระดับสูง ที่มี "นโยบาย" ว่าจะแจ้งเตือนผู้ใช้เมื่อไหร่) **รู้จัก**
`EmailSender` (โมดูลระดับล่าง ที่รู้รายละเอียดว่าจะส่งอีเมลอย่างไร) โดยตรง ปัญหา: ไม่สามารถเปลี่ยน
ช่องทางแจ้งเตือนได้โดยไม่แก้ `NotificationService` และไม่สามารถทดสอบ logic ของ
`NotificationService` แยกจากการส่งอีเมลจริงได้เลย

### เวอร์ชันที่ถูกต้อง: ทั้งสองฝั่งพึ่งพา Interface ร่วมกัน

```cpp
// dip_good.cpp
#include <iostream>
#include <memory>
#include <string>

// DIP: ทั้ง NotificationService (โมดูลระดับสูง) และ EmailSender/SmsSender
// (โมดูลระดับล่าง) ต่างพึ่งพา "นามธรรม" (IMessageSender) ร่วมกัน — ไม่มีฝ่ายไหน
// พึ่งพาอีกฝ่ายโดยตรงเลย ตรงกับหลักการ Interface ที่เรียนใน Part 50
class IMessageSender {
public:
    virtual void send(const std::string& to, const std::string& message) = 0;
    virtual ~IMessageSender() = default;
};

class EmailSender : public IMessageSender {
public:
    void send(const std::string& to, const std::string& message) override {
        std::cout << "[Email] ส่งถึง " << to << ": " << message << "\n";
    }
};

class SmsSender : public IMessageSender {
public:
    void send(const std::string& to, const std::string& message) override {
        std::cout << "[SMS] ส่งถึง " << to << ": " << message << "\n";
    }
};

class NotificationService {
public:
    // Constructor Injection: รับ "ความสามารถในการส่งข้อความ" ผ่าน interface
    // ไม่รู้จักและไม่สนใจว่าเบื้องหลังเป็น Email, SMS หรืออะไรก็ตาม
    explicit NotificationService(std::shared_ptr<IMessageSender> sender)
        : sender_(std::move(sender)) {}

    void notify(const std::string& user, const std::string& message) {
        sender_->send(user, message);
    }

private:
    std::shared_ptr<IMessageSender> sender_;
};

int main() {
    auto email_sender = std::make_shared<EmailSender>();
    NotificationService email_service(email_sender);
    email_service.notify("somchai@example.com", "ออเดอร์ของคุณจัดส่งแล้ว");

    // เปลี่ยนไปใช้ SMS แทน โดยไม่ต้องแก้โค้ด NotificationService แม้แต่บรรทัดเดียว
    auto sms_sender = std::make_shared<SmsSender>();
    NotificationService sms_service(sms_sender);
    sms_service.notify("081-234-5678", "ออเดอร์ของคุณจัดส่งแล้ว");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 dip_good.cpp -o dip_good && ./dip_good
```

```
[Email] ส่งถึง somchai@example.com: ออเดอร์ของคุณจัดส่งแล้ว
[SMS] ส่งถึง 081-234-5678: ออเดอร์ของคุณจัดส่งแล้ว
```

สังเกตว่าตอนนี้ `NotificationService` รู้จักแค่ `IMessageSender` (Abstract Class จาก Part 50) —
ไม่มีจุดไหนในโค้ดของมันที่พูดถึงคำว่า "Email" หรือ "SMS" เลยด้วยซ้ำ การเปลี่ยนช่องทางแจ้งเตือน
ทำได้แค่เปลี่ยน object ที่ inject เข้าไปตอนสร้าง — เราจะเห็นตัวอย่างที่สมบูรณ์ยิ่งขึ้นของแนวคิดนี้
พร้อมประโยชน์ด้าน Unit Testing ในหัวข้อ 114.7

### ตารางสรุป SOLID ทั้ง 5 ข้อ พร้อมตัวอย่างและ Part ที่เกี่ยวข้อง

| หลักการ | ปัญหาที่แก้ | เทคนิค C++ ที่ใช้ | อ้างอิง Part |
|---|---|---|---|
| **S**RP | คลาสทำหลายหน้าที่ที่ไม่เกี่ยวข้องกัน แก้ยาก เสี่ยงพัง | แยกคลาส/ฟังก์ชันตามความรับผิดชอบ | Part 113 (ระดับฟังก์ชัน) |
| **O**CP | เพิ่มของใหม่ต้องแก้โค้ดเดิม เสี่ยงทำของเดิมพัง | Abstract Class + Virtual Function | Part 49, Part 50 |
| **L**SP | Subclass ดูสืบทอดถูกต้อง แต่พฤติกรรมไม่ตรงสัญญา | ออกแบบ hierarchy ตามพฤติกรรมจริง ไม่ใช่แค่ความหมายทางภาษา | Part 48, Part 49 |
| **I**SP | Interface ใหญ่เกินไป บังคับ implement สิ่งที่ทำไม่ได้ | แยก Interface ย่อยตามความสามารถจริง | Part 50 |
| **D**IP | โมดูลระดับสูงผูกติดกับรายละเอียดระดับล่างโดยตรง | Constructor Injection ผ่าน Interface | Part 50, หัวข้อ 114.7 |

---

## 114.6 Layered Architecture: ประยุกต์ใช้กับ REST API จาก Part 108 (Step 910)

### แนวคิด Layered Architecture

**Layered Architecture** คือการแบ่งระบบออกเป็น "ชั้น" (layer) ที่แต่ละชั้นมีความรับผิดชอบชัดเจน
และสื่อสารกับชั้นที่อยู่ติดกันเท่านั้น รูปแบบที่นิยมที่สุดสำหรับระบบ backend มี 3 ชั้นหลัก:

| ชั้น | รับผิดชอบ | ตัวอย่างในบริบท Web API |
|---|---|---|
| **Presentation Layer** | รับ/ส่งข้อมูลกับโลกภายนอก แปลง protocol (HTTP/JSON) เป็น C++ type และกลับกัน | HTTP route handler (Crow), การ parse request/format response |
| **Business Logic Layer** | กฎทางธุรกิจล้วนๆ ไม่รู้จัก HTTP ไม่รู้จักฐานข้อมูลโดยตรง | การ validate title, การคำนวณส่วนลด, workflow ต่างๆ |
| **Data Access Layer (DAL)** | อ่าน/เขียนข้อมูลถาวร ไม่รู้จักกฎธุรกิจใดๆ | คลาสที่คุยกับ SQLite/PostgreSQL (Part 106) |

หลักการสำคัญคือ **แต่ละชั้นควรพึ่งพาแค่ชั้นที่อยู่ติดกัน และควรพึ่งพาผ่านนามธรรม (ตาม DIP) เมื่อทำได้**
Presentation Layer ไม่ควรมีกฎธุรกิจปนอยู่ และ Business Logic Layer ไม่ควรรู้จักรายละเอียดของ HTTP
หรือ SQL โดยตรง

### ย้อนดูโครงสร้างของ Task API จาก Part 108

Part 108 แบ่งโปรเจกต์ Task API ออกเป็น Model, Data Access Layer (`db.hpp`/`db.cpp`), Response
Helper, และ Routes (`task_routes.hpp`/`task_routes.cpp`) ซึ่งเป็นก้าวสำคัญจาก "ยัดทุกอย่างไว้ใน
`main.cpp`" — แต่ถ้าสังเกตให้ดี **`task_routes.cpp` ทำ 2 หน้าที่ปนกัน**: มันเป็นทั้ง Presentation
Layer (parse JSON request, สร้าง response) **และ** Business Logic Layer (ฟังก์ชัน
`validate_task_payload()` ที่มีกฎธุรกิจ เช่น "title ต้องมีความยาว 1-200 ตัวอักษร") ในไฟล์เดียวกัน

นี่ไม่ใช่ความผิดพลาดของ Part 108 — สำหรับโปรเจกต์ขนาดเล็กแบบนั้น การรวม 2 ชั้นไว้ด้วยกันเป็นการ
ตัดสินใจที่สมเหตุสมผล (ไม่ over-engineer ตั้งแต่แรก) แต่เมื่อระบบโตขึ้น กฎธุรกิจซับซ้อนขึ้น หรือ
ต้องการนำกฎธุรกิจเดียวกันไปใช้ซ้ำในหลายช่องทาง (เช่น ทั้ง REST API และ CLI tool ภายใน) การแยก
Business Logic Layer ออกมาต่างหากจะเริ่มคุ้มค่า

### Refactor: แยก Business Logic Layer ออกจาก Presentation Layer

```cpp
// layered.cpp
#include <iostream>
#include <nlohmann/json.hpp>
#include <optional>
#include <string>
#include <vector>

using json = nlohmann::json;

// =====================================================================
// ชั้นที่ 1: Data Access Layer (DAL) — เทียบเท่า db.hpp/db.cpp จาก Part 108
// รู้แค่เรื่องเดียว: เก็บและดึงข้อมูล Task ไม่รู้จัก HTTP ไม่รู้จักกฎ business ใดๆ
// (ในโค้ดจริงจาก Part 108 ชั้นนี้คุยกับ SQLite ผ่าน sqlite3 C API — ที่นี่ทำเป็น
// in-memory เพื่อให้ตัวอย่างกระชับและรันทดสอบได้โดยไม่ต้องพึ่งไฟล์ฐานข้อมูล)
struct Task {
    int id;
    std::string title;
    std::string description;
    bool done;

    json to_json() const {
        return json{{"id", id}, {"title", title}, {"description", description}, {"done", done}};
    }
};

class TaskRepository {
public:
    Task create(const std::string& title, const std::string& description) {
        Task t{next_id_++, title, description, false};
        tasks_.push_back(t);
        return t;
    }

    std::optional<Task> find_by_id(int id) const {
        for (const auto& t : tasks_) {
            if (t.id == id) return t;
        }
        return std::nullopt;
    }

private:
    std::vector<Task> tasks_;
    int next_id_ = 1;
};

// =====================================================================
// ชั้นที่ 2: Business Logic Layer — ชั้นที่ Part 108 "ไม่ได้แยกไว้ต่างหาก"
// (ใน Part 108 กฎ validate_task_payload() ถูกฝังอยู่ใน task_routes.cpp ปนกับ
// HTTP handler โดยตรง) ชั้นนี้รับผิดชอบ "กฎทางธุรกิจ" ล้วนๆ ไม่รู้จัก HTTP,
// ไม่รู้จัก JSON request/response — รับ/คืนเป็น C++ type ธรรมดา
class TaskService {
public:
    explicit TaskService(TaskRepository& repository) : repository_(repository) {}

    struct CreateResult {
        bool success;
        std::string error_message;  // มีค่าเมื่อ success == false
        Task task;                  // มีค่าเมื่อ success == true
    };

    // กฎธุรกิจ: title ห้ามว่าง และห้ามยาวเกิน 200 ตัวอักษร (ย้ายมาจาก
    // validate_task_payload() ของ Part 108 แต่แยกออกจาก HTTP โดยสมบูรณ์แล้ว)
    CreateResult create_task(const std::string& title, const std::string& description) {
        if (title.empty()) {
            return CreateResult{false, "title ห้ามว่าง", Task{}};
        }
        if (title.size() > 200) {
            return CreateResult{false, "title ต้องไม่เกิน 200 ตัวอักษร", Task{}};
        }
        Task created = repository_.create(title, description);
        return CreateResult{true, "", created};
    }

private:
    TaskRepository& repository_;
};

// =====================================================================
// ชั้นที่ 3: Presentation Layer — เทียบเท่า CROW_ROUTE handler ใน task_routes.cpp
// รู้แค่เรื่อง HTTP/JSON: แปลง request เป็น C++ type, เรียก Business Logic Layer,
// แปลงผลลัพธ์กลับเป็น JSON response — "ไม่มีกฎธุรกิจใดๆ ปนอยู่ในชั้นนี้เลย"
json handle_create_task_request(TaskService& service, const json& request_body) {
    if (!request_body.contains("title") || !request_body["title"].is_string()) {
        return json{{"success", false}, {"error", "ต้องมีฟิลด์ \"title\" เป็น string"}};
    }
    const std::string title = request_body["title"].get<std::string>();
    const std::string description = request_body.value("description", "");

    const auto result = service.create_task(title, description);
    if (!result.success) {
        return json{{"success", false}, {"error", result.error_message}};
    }
    return json{{"success", true}, {"data", result.task.to_json()}};
}

int main() {
    TaskRepository repository;              // DAL
    TaskService service(repository);        // Business Logic Layer

    // จำลอง HTTP request body ที่ Crow จะส่งมาให้ handler จริง (Presentation Layer)
    const json good_request = {{"title", "ซื้อของเข้าบ้าน"}, {"description", "นมและไข่"}};
    std::cout << handle_create_task_request(service, good_request).dump() << "\n";

    const json bad_request = {{"description", "ไม่มี title เลย"}};
    std::cout << handle_create_task_request(service, bad_request).dump() << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 layered.cpp -o layered && ./layered
```

```
{"data":{"description":"นมและไข่","done":false,"id":1,"title":"ซื้อของเข้าบ้าน"},"success":true}
{"error":"ต้องมีฟิลด์ \"title\" เป็น string","success":false}
```

ในโปรเจกต์จริงที่ต่อยอดจาก Part 108 ฟังก์ชัน `handle_create_task_request` จะถูกเรียกจากภายใน
`CROW_ROUTE(app, "/tasks").methods(crow::HTTPMethod::POST)(...)` โดยตรง — Crow ทำหน้าที่แค่
"ส่งต่อ" request body และ "ส่งกลับ" response ในรูปแบบ HTTP เท่านั้น ส่วนกฎธุรกิจทั้งหมดอยู่ใน
`TaskService` ที่**ทดสอบได้โดยไม่ต้องรัน HTTP server เลยแม้แต่ครั้งเดียว** (เขียน Unit Test ตาม
Part 93 เรียก `service.create_task(...)` ตรงๆ ได้ทันที)

### ประโยชน์ที่จับต้องได้ของการแยกชั้นแบบนี้

| ประโยชน์ | อธิบาย |
|---|---|
| ทดสอบ Business Logic โดยไม่ต้องรัน HTTP server | `TaskService::create_task()` เรียกตรงได้ใน Unit Test เร็วกว่าการยิง HTTP request จริงมาก |
| นำกฎธุรกิจไปใช้ซ้ำในหลายช่องทาง | ถ้าอนาคตมี CLI tool หรือ gRPC endpoint เพิ่ม สามารถเรียก `TaskService` เดียวกันได้ทันที ไม่ต้องเขียน validate logic ซ้ำ |
| เปลี่ยนฐานข้อมูลโดยไม่กระทบกฎธุรกิจ | เปลี่ยนจาก SQLite เป็น PostgreSQL (Part 106) แก้แค่ `TaskRepository` โดย `TaskService` ไม่ต้องแตะเลย (ตรงกับที่ Part 108 เคยกล่าวถึงไว้) |
| Reviewer เข้าใจโค้ดเร็วขึ้น | เห็นชื่อไฟล์/คลาสก็รู้ทันทีว่าควรหาอะไรที่ชั้นไหน ไม่ต้องไล่อ่านทั้งไฟล์ |

---

## 114.7 Dependency Injection ใน C++ โดยไม่มี Framework ช่วย (Step 911)

### ทำไม C++ ไม่มี Framework ฉีด Dependency ให้อัตโนมัติเหมือนภาษาอื่น

ภาษาอย่าง Java (Spring), C# (.NET's built-in DI container), หรือ TypeScript (NestJS) มี
**Dependency Injection Container** ที่จัดการการสร้างและ "ฉีด" (inject) dependency ให้อัตโนมัติผ่าน
Reflection หรือ Decorator/Annotation — โปรแกรมเมอร์แค่ประกาศว่าคลาสไหนต้องการอะไร แล้ว Framework
จะหาและประกอบร่างให้เอง

**C++ ไม่มีกลไกแบบนี้ในตัวภาษา** เพราะ C++ ไม่มี Runtime Reflection แบบเต็มรูปแบบ (ไม่รู้จัก
โครงสร้างของ class ตัวเองตอน runtime เหมือน Java/C#) ดังนั้นการทำ Dependency Injection ใน C++
จึงทำผ่าน**เทคนิคทางภาษาล้วนๆ**ที่เราต้องเขียนเอง — ข่าวดีคือเทคนิคนี้ **เรียบง่ายกว่าที่คิดมาก**
และเป็นแค่การประยุกต์ใช้ DIP (หัวข้อ 114.5) ให้เป็นระบบมากขึ้นเท่านั้น ไม่จำเป็นต้องพึ่ง Framework
ภายนอกใดๆ เลย

### Constructor Injection: รูปแบบที่นิยมที่สุดใน C++

**Constructor Injection** คือการส่ง Dependency (มักอยู่ในรูปแบบ Interface/Abstract Class ตาม DIP)
เข้าไปทาง **Constructor** ของคลาสที่ต้องการใช้มัน แทนที่จะให้คลาสนั้นสร้าง Dependency ขึ้นมาเอง
ภายใน — เราได้เห็นรูปแบบนี้มาแล้วใน `NotificationService` (หัวข้อ 114.5) มาดูตัวอย่างที่สมบูรณ์
ยิ่งขึ้น ที่แสดงประโยชน์ด้าน **การทดสอบ** อย่างชัดเจน โดยต่อยอดจาก Task API ของหัวข้อก่อนหน้า:

```cpp
// di.cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

struct Task {
    int id;
    std::string title;
};

// Interface (Part 50) ที่ Business Logic Layer พึ่งพา แทนที่จะพึ่งพา
// SQLite-backed repository ตรงๆ (ตรงตาม DIP ในหัวข้อ 114.5)
class ITaskRepository {
public:
    virtual Task create(const std::string& title) = 0;
    virtual std::size_t count() const = 0;
    virtual ~ITaskRepository() = default;
};

// Production: repository จริงที่คุยกับ SQLite (ย่อไว้ในตัวอย่างนี้ให้เป็น
// in-memory แทน — โครงสร้างเดียวกับ db.cpp ของ Part 108 ทุกประการ)
class SqliteTaskRepository : public ITaskRepository {
public:
    Task create(const std::string& title) override {
        Task t{next_id_++, title};
        tasks_.push_back(t);
        std::cout << "[SQLite] บันทึก task ลงฐานข้อมูลจริง: " << title << "\n";
        return t;
    }
    std::size_t count() const override { return tasks_.size(); }

private:
    std::vector<Task> tasks_;
    int next_id_ = 1;
};

// Test Double: ใช้เฉพาะตอนรัน Unit Test (Part 93) ไม่แตะฐานข้อมูลจริงเลย
class FakeTaskRepository : public ITaskRepository {
public:
    Task create(const std::string& title) override {
        create_call_count_++;
        return Task{999, title};
    }
    std::size_t count() const override { return create_call_count_; }
    int create_call_count() const { return create_call_count_; }

private:
    int create_call_count_ = 0;
};

// Business Logic Layer: รับ repository ผ่าน Constructor (Constructor Injection)
// ไม่สร้าง repository เองภายใน และไม่รู้ว่าเบื้องหลังเป็น SQLite หรือ Fake
class TaskService {
public:
    explicit TaskService(std::shared_ptr<ITaskRepository> repository)
        : repository_(std::move(repository)) {}

    Task create_task(const std::string& title) {
        return repository_->create(title);
    }

    std::size_t total_tasks() const { return repository_->count(); }

private:
    std::shared_ptr<ITaskRepository> repository_;
};

int main() {
    // --- Production wiring: main.cpp ประกอบร่าง service เข้ากับ repository จริง ---
    auto real_repo = std::make_shared<SqliteTaskRepository>();
    TaskService production_service(real_repo);
    production_service.create_task("ซื้อของเข้าบ้าน");
    std::cout << "จำนวน task ใน production: " << production_service.total_tasks() << "\n";

    // --- Unit Test wiring: inject FakeTaskRepository แทน ไม่แตะฐานข้อมูลจริงเลย ---
    auto fake_repo = std::make_shared<FakeTaskRepository>();
    TaskService test_service(fake_repo);
    test_service.create_task("task สำหรับทดสอบ");
    test_service.create_task("task สำหรับทดสอบอีกตัว");
    std::cout << "FakeTaskRepository.create ถูกเรียก " << fake_repo->create_call_count()
              << " ครั้ง (ยืนยันพฤติกรรมโดยไม่ต้องมี SQLite จริง)\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 di.cpp -o di && ./di
```

```
[SQLite] บันทึก task ลงฐานข้อมูลจริง: ซื้อของเข้าบ้าน
จำนวน task ใน production: 1
FakeTaskRepository.create ถูกเรียก 2 ครั้ง (ยืนยันพฤติกรรมโดยไม่ต้องมี SQLite จริง)
```

จุดที่ต้องสังเกตให้ชัด: `TaskService` **เขียนขึ้นเพียงครั้งเดียว** แต่ถูกใช้งานได้ทั้งใน production
(กับ `SqliteTaskRepository`) และใน unit test (กับ `FakeTaskRepository`) โดย**ไม่มีการแก้โค้ดของ
`TaskService` เลยแม้แต่บรรทัดเดียว** — จุดที่ตัดสินใจว่าจะ "ประกอบร่าง" (wire) service เข้ากับ
repository ตัวไหน เกิดขึ้นแค่ที่เดียวเท่านั้น: **ใน `main()` (สำหรับ production) หรือในไฟล์ test
(สำหรับ unit test)** จุดนี้เรียกว่า **Composition Root** — เป็นจุดเดียวในโปรแกรมทั้งหมดที่ "รู้จัก"
ทุก concrete class และประกอบร่างพวกมันเข้าด้วยกัน ส่วนที่เหลือของโปรแกรมรู้จักแค่ interface เท่านั้น

### เปรียบเทียบ Dependency Injection แบบต่างๆ

| รูปแบบ | วิธีการ | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **Constructor Injection** | ส่ง dependency ผ่าน constructor | บังคับให้ dependency ครบก่อนใช้งานได้ (compile-time safety) ชัดเจนที่สุดว่าคลาสต้องการอะไรบ้าง | Constructor อาจมีพารามิเตอร์เยอะถ้า dependency เยอะ |
| **Setter Injection** | มีเมธอด `set_xxx()` แยกต่างหากสำหรับกำหนด dependency ภายหลัง | ยืดหยุ่นกว่า เปลี่ยน dependency ได้หลังสร้าง object แล้ว | เสี่ยงลืมเรียก setter แล้วใช้งาน object ทั้งที่ dependency ยังไม่ถูกกำหนด (runtime error แทน compile-time) |
| **Service Locator** | คลาสขอ dependency จาก "ที่เก็บกลาง" เอง ณ จุดที่ต้องใช้ | ไม่ต้องส่งผ่าน constructor ยาวๆ | ซ่อน dependency ที่แท้จริงไว้ ทำให้อ่านโค้ดแล้วไม่รู้ว่าคลาสต้องการอะไรบ้างจนกว่าจะไล่อ่าน implementation ทั้งหมด (ถือเป็น Anti-pattern ในหลายกรณี) |

> **คำแนะนำของ Part นี้**: ใช้ **Constructor Injection** เป็นค่าเริ่มต้นเสมอ เพราะทำให้ dependency
> ที่คลาสต้องการปรากฏชัดเจนใน signature ของ constructor — ใครอ่านโค้ดแค่เปิดดู constructor ก็รู้
> ทันทีว่าคลาสนี้ต้องพึ่งพาอะไรบ้าง โดยไม่ต้องไล่อ่านทุกเมธอดเพื่อค้นหา

---

## 114.8 Technical Debt: เมื่อไหร่ควร Refactor เมื่อไหร่ควร Rewrite (Step 912)

### Technical Debt คืออะไร

**Technical Debt** (หนี้ทางเทคนิค) เป็นคำอุปมาที่ Ward Cunningham (หนึ่งในผู้ร่วมเขียน Agile
Manifesto) บัญญัติขึ้นในปี 1992 เพื่ออธิบายผลลัพธ์ของการเลือกวิธีแก้ปัญหาที่ **เร็วแต่ไม่ยั่งยืน**
แทนวิธีที่ถูกต้องแต่ใช้เวลานานกว่า — เหมือนการกู้เงิน: ได้ผลลัพธ์เร็วในวันนี้ แต่ต้อง "จ่ายดอกเบี้ย"
ในอนาคตในรูปของความยากลำบากที่เพิ่มขึ้นทุกครั้งที่ต้องแก้ไขหรือต่อยอดโค้ดส่วนนั้น

### ประเภทของ Technical Debt (Technical Debt Quadrant)

Martin Fowler เสนอกรอบคิดที่แบ่ง Technical Debt ออกเป็น 4 ประเภทตาม 2 มิติ: **ตั้งใจหรือไม่ตั้งใจ**
และ **มีเหตุผลรองรับหรือไม่**

| | **ตั้งใจ (Deliberate)** | **ไม่ตั้งใจ (Inadvertent)** |
|---|---|---|
| **มีเหตุผลรองรับ (Prudent)** | "เรารู้ว่าโค้ดนี้ไม่สมบูรณ์แบบ แต่ deadline บีบ ต้องส่งก่อน แล้วค่อยกลับมาแก้" | "ตอนนี้เราถึงจะรู้ว่าควรออกแบบแบบนี้ตั้งแต่แรก" (เกิดจากการเรียนรู้ตามธรรมชาติ) |
| **ไม่มีเหตุผลรองรับ (Reckless)** | "ไม่มีเวลาออกแบบ ทำไปก่อนไม่ต้องคิดมาก" โดยไม่ประเมินผลกระทบ | "SOLID คืออะไร?" (ขาดความรู้พื้นฐาน ไม่ใช่การตัดสินใจอย่างมีข้อมูล) |

จุดสำคัญคือ **Technical Debt แบบ "Prudent + Deliberate" ไม่ใช่เรื่องผิด** — ทีมที่มีวุฒิภาวะทาง
วิศวกรรมสามารถตัดสินใจ "กู้หนี้ทางเทคนิค" อย่างมีสติได้ ตราบใดที่**บันทึกไว้ชัดเจน** (เช่น เขียน
`// TODO: ควร refactor เป็น Strategy Pattern ถ้ามีวิธีคำนวณส่วนลดเพิ่มเกิน 3 แบบ` หรือสร้าง
Issue/Ticket ติดตาม) และ**วางแผนจะ "จ่ายคืน"** ในภายหลัง สิ่งที่อันตรายคือ Technical Debt แบบ
"Reckless" ที่สะสมโดยไม่มีใครรู้ตัวจนกระทั่งการเพิ่มฟีเจอร์ใหม่แต่ละครั้งช้าลงเรื่อยๆ

### สัญญาณที่บอกว่า Technical Debt เริ่มส่งผลกระทบร้ายแรง

- ทีมใช้เวลา "ทำความเข้าใจโค้ดเดิม" นานกว่าเวลาที่ใช้เขียนโค้ดใหม่จริง
- ฟีเจอร์เล็กๆ ที่ควรใช้เวลา 1 วัน กลับใช้เวลา 1 สัปดาห์ เพราะต้องแก้ไขหลายจุดที่เชื่อมโยงกันแบบ
  คาดเดาไม่ได้ (สัญญาณของการละเมิด SRP/DIP อย่างกว้างขวาง)
- วิศวกรใหม่ในทีมใช้เวลานานผิดปกติกว่าจะเข้าใจ codebase และเริ่มทำงานได้อย่างมีประสิทธิภาพ
- บั๊กเดิมที่เคยแก้ไปแล้วกลับมาเกิดซ้ำในรูปแบบใหม่ (สัญญาณของโค้ดซ้ำซ้อนที่ไม่ได้ถูกรวมเป็นจุดเดียว)

### กรอบการตัดสินใจ: Refactor หรือ Rewrite

คำถามที่ทุกทีมต้องเผชิญเมื่อ Technical Debt สะสมมากคือ: **"ควรค่อยๆ ปรับปรุงโค้ดเดิม (Refactor)
หรือควรเขียนใหม่ทั้งหมด (Rewrite)?"** นี่คือหนึ่งในการตัดสินใจที่มีความเสี่ยงสูงที่สุดในวงการซอฟต์แวร์
— มีตัวอย่างชื่อดังหลายกรณีที่บริษัทเลือก Rewrite ทั้งระบบแล้วใช้เวลานานกว่าที่คาดมาก จนเกือบทำให้
ธุรกิจล้มเหลว (กรณีศึกษาที่มีชื่อเสียงที่สุดคือบทความ "Things You Should Never Do" ของ Joel
Spolsky ปี 2000 ที่เตือนเรื่องการ Rewrite Netscape ทั้งหมดจนเสียส่วนแบ่งตลาดไปอย่างมาก)

**ตารางเปรียบเทียบเพื่อช่วยตัดสินใจ**

| ปัจจัย | เอียงไปทาง Refactor | เอียงไปทาง Rewrite |
|---|---|---|
| ขนาดของปัญหา | ปัญหาอยู่ในบางส่วนของระบบ (เช่น 1-2 โมดูล) | ปัญหาอยู่ในสถาปัตยกรรมพื้นฐานทั้งหมด (เช่น เลือกใช้ภาษา/library ผิดตั้งแต่ต้น) |
| Test Coverage ปัจจุบัน | มี Unit Test คุ้มครองพฤติกรรมเดิมอยู่แล้ว (Part 93) | แทบไม่มี test เลย ทำให้ "รู้ว่าอะไรควรทำงานอย่างไร" ยากอยู่แล้ว |
| ธุรกิจยังต้องการฟีเจอร์ใหม่ระหว่างทางหรือไม่ | ต้องการ (Refactor ทำควบคู่กับพัฒนาฟีเจอร์ใหม่ได้) | ถ้า Rewrite ทั้งหมด มักต้อง "หยุด" พัฒนาฟีเจอร์ใหม่บนระบบเดิมเป็นเวลานาน |
| ความเข้าใจใน Business Logic เดิม | เข้าใจดี มีเอกสาร/คนที่รู้ระบบอยู่ | Business Logic เดิมมีรายละเอียดซ่อนอยู่มาก (Edge case ที่สะสมมาหลายปี) ซึ่ง Rewrite เสี่ยงตกหล่น |
| ระยะเวลาที่ยอมรับความเสี่ยงได้ | ต้องการความเสี่ยงต่ำ ส่งมอบสม่ำเสมอ | มีทรัพยากรและเวลาเพียงพอที่จะยอมรับความเสี่ยงระยะยาว |

> **หลักการที่ปลอดภัยที่สุดในทางปฏิบัติ**: เลือก **Refactor แบบค่อยเป็นค่อยไป** เป็นตัวเลือกแรกเสมอ
> โดยใช้เทคนิค **Strangler Fig Pattern** — ค่อยๆ สร้างส่วนใหม่ที่ดีกว่าขึ้นมาทดแทนส่วนเก่าทีละส่วน
> (คล้ายต้นไม้ Strangler Fig ที่ค่อยๆ เติบโตพันรอบต้นไม้เดิมจนแทนที่ทั้งหมดในที่สุด) โดยระบบเก่าและ
> ใหม่**ทำงานคู่ขนานกันได้ระหว่างการเปลี่ยนผ่าน** วิธีนี้ลดความเสี่ยงกว่าการ "หยุดทุกอย่างแล้วเขียน
> ใหม่หมดในคราวเดียว" อย่างมาก และสอดคล้องกับ Layered Architecture ในหัวข้อ 114.6 พอดี — เพราะ
> ถ้าระบบถูกแบ่งเป็นชั้นที่ชัดเจนอยู่แล้ว การแทนที่ทีละชั้น (เช่น เปลี่ยน Data Access Layer ก่อน
> โดยที่ Business Logic Layer ไม่ต้องแตะ) ทำได้ปลอดภัยกว่าระบบที่ทุกอย่างพันกันยุ่งเหยิงมาก

**Rewrite ทั้งหมดควรพิจารณาก็ต่อเมื่อ** ปัญหาอยู่ในระดับรากฐาน (เช่น เลือกภาษาผิดสำหรับงานที่ต้องการ
Performance ขนาดนี้, สถาปัตยกรรมพื้นฐานขัดแย้งกับความต้องการใหม่โดยสิ้นเชิง) **และ** ทีมมีความมั่นใจ
สูงว่าเข้าใจ Business Logic เดิมครบถ้วน (ผ่าน Test Coverage ที่ดีหรือเอกสารที่ครบถ้วน) **และ** ธุรกิจ
สามารถรับความเสี่ยงของการหยุดพัฒนาฟีเจอร์ใหม่ชั่วคราวได้จริง — ถ้าเงื่อนไขใดเงื่อนไขหนึ่งไม่ครบ
ควรเลือก Refactor แบบค่อยเป็นค่อยไปแทนเสมอ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **มองว่า SOLID เป็นกฎที่ต้องใช้ครบทุกข้อในทุกคลาส** — SOLID เป็นเครื่องมือช่วยคิดเวลาที่ระบบ
   เริ่มซับซ้อนและต้องเปลี่ยนแปลงบ่อย ไม่ใช่กฎที่ต้องบังคับใช้กับ struct ง่ายๆ ที่ไม่มีทีท่าว่าจะ
   เปลี่ยนแปลง การใส่ Interface และ Dependency Injection ให้กับโค้ดที่ไม่เคยต้องการความยืดหยุ่น
   เลยคือการ Over-Engineer (ย้ำเตือนจาก Part 80 เช่นกัน)
2. **สร้าง Abstract Class/Interface ล่วงหน้าโดยไม่มี concrete implementation ตัวที่สองจริงๆ** —
   ถ้าระบบมี `IMessageSender` แต่มี `EmailSender` เป็น implementation เดียวตลอดอายุโปรเจกต์
   และไม่มีแผนจะเพิ่มตัวที่สองเลย การสร้าง interface ล่วงหน้าอาจเป็นความซับซ้อนที่ไม่จำเป็น —
   หลักการที่ปลอดภัยกว่าคือสร้าง abstraction เมื่อ**เห็นความจำเป็นจริงแล้ว** (เช่น ต้องการ inject
   Mock สำหรับ Unit Test อยู่แล้ว ซึ่งนับเป็นเหตุผลที่เพียงพอ)
3. **เข้าใจผิดว่า Inheritance ทุกกรณีคือ OOP ที่ดี** — ตัวอย่าง Rectangle/Square พิสูจน์ชัดว่า
   ความสัมพันธ์แบบ "is-a" ทางภาษาไม่ได้แปลว่า "แทนที่กันได้จริง" เสมอไป ต้องตรวจสอบพฤติกรรม
   ไม่ใช่แค่ไวยากรณ์
4. **แยก Layer ตามชื่อไฟล์ แต่ logic ยังปนกันจริงในทางปฏิบัติ** — เช่น สร้างไฟล์ `service.cpp`
   แยกออกมา แต่ภายในยังคง `#include <crow.h>` และสร้าง `crow::response` ตรงๆ อยู่ดี ทำให้ยังผูก
   กับ HTTP framework อยู่เหมือนเดิม การแยก Layer ที่แท้จริงต้องดูที่ **การพึ่งพา (dependency)**
   ไม่ใช่แค่ที่ตั้งของไฟล์
5. **สร้างหนี้ทางเทคนิคแบบ "Reckless" โดยไม่รู้ตัวว่ากำลังสร้างหนี้อยู่** — ต่างจากหนี้แบบมีเหตุผล
   รองรับ (Prudent) การเขียนโค้ดลวกๆ โดยไม่รู้ว่ามีทางที่ดีกว่า (เพราะขาดความรู้พื้นฐานอย่าง SOLID)
   เป็นสิ่งที่ป้องกันได้ด้วยการเรียนรู้และ Code Review (Part 113) ที่จริงจัง
6. **ตัดสินใจ Rewrite ทั้งระบบเพราะ "โค้ดเก่าดูน่าเบื่อ" โดยไม่ประเมินความเสี่ยงอย่างเป็นระบบ** —
   ดังที่กล่าวไว้ในหัวข้อ 114.8 การ Rewrite มีความเสี่ยงสูงกว่าที่คิดมาก ควรใช้ตารางเปรียบเทียบ
   ในหัวข้อนั้นประเมินอย่างจริงจังก่อนตัดสินใจ ไม่ใช่ตัดสินใจจากความรู้สึกอยากเขียนโค้ดใหม่เพียงอย่างเดียว
7. **ใช้ `std::shared_ptr` สำหรับ Dependency Injection ทุกกรณีโดยไม่คิด** — ในตัวอย่างของ Part นี้
   ใช้ `std::shared_ptr<IMessageSender>` เพื่อความง่าย แต่ในโค้ดจริงควรพิจารณาว่าจำเป็นต้องแชร์
   ownership จริงหรือไม่ ถ้า dependency มีอายุยาวกว่าคลาสที่ใช้มันเสมอ (เช่น อยู่ตลอดอายุโปรแกรม)
   การรับเป็น reference (`IMessageSender&`) อาจเหมาะสมกว่าและมี overhead น้อยกว่า (อ้างอิงหลักการ
   Rule of Zero/Ownership จาก Part 80)

---

## แบบฝึกหัดท้ายบท

1. หยิบตัวอย่าง `SalesReport` (SRP ที่ไม่ดี) ในหัวข้อ 114.1 มาเพิ่มความสามารถ "ส่งรายงานเป็นอีเมล"
   เข้าไป ทำใน**ทั้งสองเวอร์ชัน** (ก่อนและหลัง Refactor) แล้วเปรียบเทียบว่าเวอร์ชันไหนเพิ่มความ
   สามารถนี้ได้ง่ายกว่าและเสี่ยงทำของเดิมพังน้อยกว่า

2. จากตัวอย่าง OCP (หัวข้อ 114.2) ให้เพิ่มรูปทรง `Pentagon` (ห้าเหลี่ยมด้านเท่า) เข้าไปในระบบ
   โดย**ห้ามแก้ไข**ฟังก์ชัน `calculate_total_area()` แม้แต่บรรทัดเดียว (ดูแนวทางเฉลยข้อ 1 ด้านล่าง
   ที่ใช้ตัวอย่างคล้ายกันกับระบบชำระเงิน)

3. หาโค้ดของตัวเองที่เคยเขียนในโปรเจกต์ก่อนหน้าของหลักสูตรนี้ (เลือกคลาสหรือฟังก์ชันมา 1 จุด) แล้ว
   ตรวจสอบว่าละเมิดหลักการ SOLID ข้อใดข้อหนึ่งหรือไม่ ถ้าพบ ให้ Refactor ตามหลักการที่เรียนมา

4. เขียน Interface `ILogger` ที่มีเมธอด `log(const std::string& message)` แล้วสร้าง
   implementation 2 ตัว: `ConsoleLogger` (print ออกหน้าจอ) และ `FileLogger` (เขียนลงไฟล์) จากนั้น
   เขียนคลาส `OrderProcessor` ที่รับ `ILogger` ผ่าน Constructor Injection และเรียก `log()` ตอน
   ประมวลผลออเดอร์สำเร็จ (ดูแนวทางเฉลยข้อ 2 ด้านล่าง)

5. อธิบาย (เขียนเป็นข้อความ) ว่าทำไมตัวอย่าง `IMultiFunctionPrinter` ในหัวข้อ 114.4 ถึงละเมิดทั้ง
   Interface Segregation Principle **และ** มีกลิ่นของการละเมิด Liskov Substitution Principle
   ไปพร้อมกัน (ใบ้: ย้อนกลับไปดูสัญญาณเตือนในหัวข้อ 114.3 ท้ายสุด)

6. สมมติทีมของคุณกำลังดูแลระบบที่มี Technical Debt สูงมาก (ฟีเจอร์เล็กๆ ใช้เวลาแก้ไขนานผิดปกติ
   ไม่มี Unit Test เลย) ให้เขียนแผนการตัดสินใจของตัวเอง โดยใช้ตารางเปรียบเทียบในหัวข้อ 114.8
   ประกอบ ว่าจะเลือก Refactor หรือ Rewrite พร้อมให้เหตุผลอย่างน้อย 3 ข้อ

### แนวทางเฉลยข้อ 2

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

// โจทย์: ระบบชำระเงินเดิมมี PaymentMethod แบบ abstract อยู่แล้ว (ปิดต่อการแก้ไข)
// ให้เพิ่มวิธีชำระเงินใหม่ "QRPayment" โดยห้ามแก้ checkout() แม้แต่บรรทัดเดียว
class PaymentMethod {
public:
    virtual void pay(double amount) const = 0;
    virtual ~PaymentMethod() = default;
};

class CreditCardPayment : public PaymentMethod {
public:
    void pay(double amount) const override {
        std::cout << "ชำระ " << amount << " บาท ผ่านบัตรเครดิต\n";
    }
};

class PayPalPayment : public PaymentMethod {
public:
    void pay(double amount) const override {
        std::cout << "ชำระ " << amount << " บาท ผ่าน PayPal\n";
    }
};

// checkout() ไม่ต้องแก้ไขเลยไม่ว่าจะเพิ่มวิธีชำระเงินใหม่กี่แบบก็ตาม
void checkout(const PaymentMethod& method, double amount) {
    std::cout << "กำลังดำเนินการชำระเงิน...\n";
    method.pay(amount);
    std::cout << "ชำระเงินสำเร็จ\n";
}

// ----- คำตอบ: เพิ่ม QRPayment โดยไม่แตะ checkout() เลย -----
class QRPayment : public PaymentMethod {
public:
    void pay(double amount) const override {
        std::cout << "ชำระ " << amount << " บาท ผ่าน QR Code พร้อมเพย์\n";
    }
};

int main() {
    std::vector<std::unique_ptr<PaymentMethod>> methods;
    methods.push_back(std::make_unique<CreditCardPayment>());
    methods.push_back(std::make_unique<PayPalPayment>());
    methods.push_back(std::make_unique<QRPayment>());

    for (const auto& m : methods) {
        checkout(*m, 250.0);
    }
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex1_ocp.cpp -o ex1_ocp && ./ex1_ocp
```

```
กำลังดำเนินการชำระเงิน...
ชำระ 250 บาท ผ่านบัตรเครดิต
ชำระเงินสำเร็จ
กำลังดำเนินการชำระเงิน...
ชำระ 250 บาท ผ่าน PayPal
ชำระเงินสำเร็จ
กำลังดำเนินการชำระเงิน...
ชำระ 250 บาท ผ่าน QR Code พร้อมเพย์
ชำระเงินสำเร็จ
```

**อธิบาย**: `QRPayment` เป็นคลาสใหม่ที่ implement `PaymentMethod::pay()` เหมือนกับ
`CreditCardPayment` และ `PayPalPayment` ทุกประการ `checkout()` เรียก `method.pay(amount)` ผ่าน
Dynamic Dispatch โดยไม่รู้เลยว่ากำลังคุยกับ implementation ตัวไหน — สังเกตว่าฟังก์ชัน `checkout()`
เขียนขึ้นครั้งเดียวตั้งแต่ต้นบทเรียนและไม่เคยถูกแก้ไขอีกเลยตลอดทั้งแบบฝึกหัดนี้ นี่คือ Open-Closed
Principle ทำงานได้ตามที่ออกแบบไว้อย่างสมบูรณ์

### แนวทางเฉลยข้อ 4

```cpp
#include <fstream>
#include <iostream>
#include <memory>
#include <string>

// ILogger: interface กลางที่ business logic พึ่งพา แทนที่จะผูกกับปลายทาง log ที่เจาะจง
class ILogger {
public:
    virtual void log(const std::string& message) = 0;
    virtual ~ILogger() = default;
};

class ConsoleLogger : public ILogger {
public:
    void log(const std::string& message) override {
        std::cout << "[Console] " << message << "\n";
    }
};

class FileLogger : public ILogger {
public:
    explicit FileLogger(const std::string& path) : path_(path) {}
    void log(const std::string& message) override {
        std::ofstream out(path_, std::ios::app);
        out << message << "\n";
    }

private:
    std::string path_;
};

// OrderProcessor: ธุรกิจไม่รู้จักและไม่สนใจว่า log ไปที่ไหน (Constructor Injection)
class OrderProcessor {
public:
    explicit OrderProcessor(std::shared_ptr<ILogger> logger) : logger_(std::move(logger)) {}

    void process_order(int order_id) {
        // ... สมมติ logic ประมวลผลออเดอร์จริงอยู่ตรงนี้ ...
        logger_->log("ประมวลผลออเดอร์ #" + std::to_string(order_id) + " สำเร็จ");
    }

private:
    std::shared_ptr<ILogger> logger_;
};

int main() {
    auto console_logger = std::make_shared<ConsoleLogger>();
    OrderProcessor processor_with_console(console_logger);
    processor_with_console.process_order(101);

    auto file_logger = std::make_shared<FileLogger>("/tmp/order_log_scratch_test.txt");
    OrderProcessor processor_with_file(file_logger);
    processor_with_file.process_order(102);

    std::cout << "ประมวลผลเสร็จทั้งสองแบบ โดย OrderProcessor ไม่มีจุดใดรู้จัก "
                 "ConsoleLogger หรือ FileLogger โดยตรงเลย\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex4_di.cpp -o ex4_di && ./ex4_di
```

```
[Console] ประมวลผลออเดอร์ #101 สำเร็จ
ประมวลผลเสร็จทั้งสองแบบ โดย OrderProcessor ไม่มีจุดใดรู้จัก ConsoleLogger หรือ FileLogger โดยตรงเลย
```

**อธิบาย**: `OrderProcessor` รับ `ILogger` ผ่าน constructor โดยไม่สนใจว่าเบื้องหลังจะ log ไปที่
console หรือไฟล์ — สังเกตว่า `process_order(102)` เขียน log ลงไฟล์จริงที่
`/tmp/order_log_scratch_test.txt` โดยไม่มีข้อความปรากฏบนหน้าจอ (เพราะไม่ใช่ `ConsoleLogger`)
ในขณะที่ `process_order(101)` แสดงผลบนหน้าจอทันที นี่คือตัวอย่างที่สมบูรณ์ของ Constructor
Injection ที่ทำให้คลาสเดียวกันทำงานร่วมกับ dependency คนละตัวได้อย่างอิสระ โดยไม่ต้องแก้โค้ด
`OrderProcessor` เลยแม้แต่บรรทัดเดียว — หลักการเดียวกันนี้สามารถนำไปใช้ในการเขียน Unit Test
(Part 93) ได้ทันทีด้วยการสร้าง `MockLogger` ที่บันทึกข้อความไว้ตรวจสอบ แทนที่จะ log ไปที่ปลายทาง
จริงระหว่างการทดสอบ

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เรียนรู้ **SOLID Principles** ทั้ง 5 ข้อผ่านตัวอย่างโค้ด C++ จริงที่คอมไพล์และรันได้ทุกตัว
  ตั้งแต่ Single Responsibility, Open-Closed (เชื่อมโยงกับ Abstract Class จาก Part 50), Liskov
  Substitution (พิสูจน์ด้วย assertion ที่ล้มเหลวจริง), Interface Segregation, ไปจนถึง
  Dependency Inversion
- ออกแบบและ implement **Layered Architecture** (Presentation / Business Logic / Data Access)
  โดยใช้ Task API จาก **Part 108** เป็นกรณีศึกษาจริง เห็นวิธีแยก Business Logic Layer ที่เคยปน
  อยู่กับ HTTP handler ออกมาต่างหาก
- ทำ **Dependency Injection** แบบ Constructor Injection ด้วยมือใน C++ โดยไม่พึ่ง Framework
  ภายนอก พร้อมเข้าใจแนวคิด **Composition Root** และเห็นประโยชน์ด้านการทดสอบอย่างเป็นรูปธรรม
- เข้าใจแนวคิด **Technical Debt** ทั้ง 4 ประเภทตาม Technical Debt Quadrant และได้กรอบการ
  ตัดสินใจที่เป็นระบบสำหรับเลือกระหว่าง **Refactor** แบบค่อยเป็นค่อยไป (Strangler Fig Pattern)
  กับ **Rewrite** ทั้งหมด

หลักการทั้งหมดใน Part นี้ต่อยอดโดยตรงจาก Clean Code ใน Part 113 — ถ้า Part 113 สอนให้เขียน
"ฟังก์ชันและคลาสแต่ละตัว" ให้ดี Part นี้สอนให้มองภาพรวมว่า **หลายๆ คลาสควรประกอบกันเป็นระบบ
อย่างไร** ให้ยืดหยุ่นต่อการเปลี่ยนแปลงในระยะยาว ทั้งสองระดับนี้ทำงานร่วมกันเพื่อสร้างซอฟต์แวร์ที่
ทั้ง **อ่านง่ายในระดับรายละเอียด** และ **ปรับตัวได้ในระดับภาพรวม**

ใน **Part 115** เราจะเปลี่ยนมุมมองไปที่ **Security** — นำความรู้เรื่อง Memory Safety ที่สะสมมา
ตลอดหลักสูตร (Buffer Overflow จาก Part 7, Use-After-Free จาก Part 38/96, Integer Overflow จาก
Part 89) มามองในมุม**ช่องโหว่ด้านความปลอดภัย**อย่างจริงจัง พร้อมทดสอบจริงว่า Compiler Flag อย่าง
Stack Protector และ `_FORTIFY_SOURCE` ช่วยป้องกันการโจมตีได้จริงแค่ไหน

**ต่อไป:** [Part 115 — Security ใน C/C++: ช่องโหว่ที่พบบ่อยและ Secure Coding Practice](./part-115-security-cpp.md)
