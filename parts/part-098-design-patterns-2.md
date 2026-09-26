# Part 98: Design Pattern ใน C++ (ตอนที่ 2): Structural และ Behavioral Patterns (Step 777–784)

> Module H — Build Systems, Testing และ Tooling | Part 98 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 777–784
> Part ก่อนหน้า: [Part 97 — Design Pattern ใน C++ (ตอนที่ 1): Creational Patterns](./part-097-design-patterns-1.md) | Part ถัดไป: [Part 99 — ภาพรวม Web Development ด้วย C/C++](./part-099-web-dev-overview.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายจุดประสงค์ของ **Structural Pattern** (การประกอบโครงสร้าง) และ **Behavioral
   Pattern** (การสื่อสาร/พฤติกรรม) และบอกความแตกต่างจาก Creational Pattern ที่เรียนไปแล้ว
2. ใช้ **Adapter Pattern** แปลง interface เก่าที่แก้ไขไม่ได้ให้ทำงานร่วมกับโค้ดใหม่ได้
3. ใช้ **Decorator Pattern** เพิ่มพฤติกรรมให้ object แบบ dynamic ที่ runtime โดยไม่ต้องสร้าง
   subclass ทุกชุดค่าผสม
4. ใช้ **Facade Pattern** ซ่อนความซับซ้อนของ subsystem หลายตัวไว้หลัง interface เดียวที่
   ใช้งานง่าย
5. ใช้ **Composite Pattern** จัดการโครงสร้างต้นไม้ (tree) ที่มีทั้ง "หน่วยเดี่ยว" และ "กลุ่ม"
   ผ่าน interface เดียวกัน
6. ใช้ **Observer Pattern** สร้างระบบแจ้งเตือนแบบ publish-subscribe โดยใช้ `std::weak_ptr`
   ป้องกันปัญหา memory leak และ dangling pointer — พื้นฐานสำคัญก่อนเข้า Module I
7. ใช้ **Strategy Pattern** สลับพฤติกรรมของ algorithm ที่ runtime ทั้งแบบ class hierarchy และ
   แบบ `std::function`/lambda สมัยใหม่
8. ใช้ **Command Pattern** ห่อคำสั่งเป็น object เพื่อรองรับ undo/redo และคิวคำสั่ง
9. implement **Iterator Pattern** ของตัวเองที่เข้ากันได้กับ range-based for loop และ STL
   algorithm โดยเชื่อมโยงกับแนวคิด iterator จาก Part 62
10. ประเมินได้ว่าเมื่อไหร่ "ควร" และ "ไม่ควร" ใช้ Design Pattern เพื่อหลีกเลี่ยงการ
    over-engineer โค้ดที่ควรจะเรียบง่าย

---

## 98.1 Structural Pattern คืออะไร + Adapter Pattern (Step 777)

### Structural Pattern คืออะไร

Creational Pattern ใน Part 97 ตอบคำถาม "จะ**สร้าง** object อย่างไร" ส่วน **Structural
Pattern** ตอบคำถามถัดไป: เมื่อมี object/class หลายตัวแล้ว จะ **ประกอบ (compose)** พวกมัน
เข้าด้วยกันเป็นโครงสร้างที่ใหญ่ขึ้นอย่างไร ให้ยังคงยืดหยุ่น แก้ไขง่าย และไม่ผูกติดกันแน่นเกินไป
(low coupling) หนังสือ GoF มี Structural Pattern ทั้งหมด 7 แบบ บทนี้จะเจาะลึก 4 แบบที่ใช้
บ่อยที่สุดในโค้ด C++ จริง: **Adapter, Decorator, Facade, Composite**

### ปัญหาที่ Adapter Pattern แก้

ในโลกจริง เรามักต้องทำงานกับ library เก่า, ระบบ legacy, หรือ third-party library ที่มี
interface **ไม่ตรงกับที่โค้ดส่วนที่เหลือของเราคาดหวัง** และเราแก้ไข source code ของมันไม่ได้
(อาจเพราะเป็น library แบบ binary หรือแก้แล้วจะกระทบระบบอื่น) **Adapter Pattern** แก้ปัญหานี้
โดยสร้าง class ตัวกลางที่ **"แปล" การเรียกจาก interface ใหม่ ไปเป็นการเรียก interface เก่า**

```
┌────────────┐            ┌──────────────────────┐          ┌───────────────┐
│   Client    │──ใช้ผ่าน──▷│     Printer            │◁─implement─│LegacyPrinterAdapter│
│ (โค้ดใหม่) │            │  (Target interface)   │          ├───────────────┤
└────────────┘            │ +print(string)*        │          │-legacy: LegacyPrinter│
                           └──────────────────────┘          │+print(string)  │
                                                              │  (เรียก legacy_->printOldStyle())│
                                                              └───────┬───────┘
                                                                      │ ห่อ (wrap)
                                                              ┌───────▼───────┐
                                                              │ LegacyPrinter  │ (interface เก่า
                                                              │+printOldStyle(const char*)│ แก้ไขไม่ได้)
                                                              └───────────────┘
```

```cpp
#include <iostream>
#include <memory>
#include <string>

// ---------- Legacy interface: ระบบเก่าที่แก้ไขไม่ได้ (เช่นมาจาก library ภายนอก) ----------
class LegacyPrinter {
public:
    void printOldStyle(const char* text) const {
        std::cout << "[LegacyPrinter] " << text << '\n';
    }
};

// ---------- Target interface: interface ใหม่ที่โค้ดส่วนที่เหลือของระบบคาดหวัง ----------
class Printer {
public:
    virtual ~Printer() = default;
    virtual void print(const std::string& text) const = 0;
};

// ---------- Adapter: แปลง interface เก่าให้ตรงกับ interface ใหม่ ----------
class LegacyPrinterAdapter : public Printer {
public:
    explicit LegacyPrinterAdapter(std::unique_ptr<LegacyPrinter> legacy)
        : legacy_(std::move(legacy)) {}

    void print(const std::string& text) const override {
        legacy_->printOldStyle(text.c_str());   // แปลง std::string เป็น const char* ให้เอง
    }

private:
    std::unique_ptr<LegacyPrinter> legacy_;
};

// ---------- Concrete Printer สมัยใหม่ (ไม่ต้องใช้ adapter) ----------
class ModernPrinter : public Printer {
public:
    void print(const std::string& text) const override {
        std::cout << "[ModernPrinter] " << text << '\n';
    }
};

// ---------- Client: ใช้งานผ่าน interface เดียว ไม่สนใจว่าข้างในเป็นของเก่าหรือของใหม่ ----------
void printDocument(const Printer& printer, const std::string& text) {
    printer.print(text);
}

int main() {
    ModernPrinter modern;
    printDocument(modern, "รายงานประจำเดือน");

    LegacyPrinterAdapter adapter(std::make_unique<LegacyPrinter>());
    printDocument(adapter, "ใบเสร็จรับเงินจากระบบเก่า");
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 adapter.cpp -o adapter
./adapter
```

ผลลัพธ์:

```
[ModernPrinter] รายงานประจำเดือน
[LegacyPrinter] ใบเสร็จรับเงินจากระบบเก่า
```

จุดสำคัญ: `printDocument()` (client) รู้จักแค่ interface `Printer` เท่านั้น ไม่รู้เลยว่าเบื้องหลัง
`LegacyPrinterAdapter` กำลังคุยกับ `LegacyPrinter` แบบเก่าอยู่ — Adapter ทำให้ระบบเก่ากับ
ระบบใหม่ **อยู่ร่วมกันได้โดยไม่ต้องแก้โค้ดฝั่งใดฝั่งหนึ่งเลย** นี่คือเหตุผลที่ Adapter ถูกใช้บ่อย
มากตอน migrate ระบบเก่าไปทีละส่วน (**Strangler Fig Pattern** ในระดับสถาปัตยกรรมใหญ่ก็ใช้
แนวคิดเดียวกันนี้)

---

## 98.2 Decorator Pattern (Step 778)

### ปัญหาที่ Decorator แก้: Subclass Explosion

ลองจินตนาการร้านกาแฟที่มีเครื่องดื่มพื้นฐาน (Espresso) และ topping เสริม (นม, น้ำตาล,
วิปครีม) ที่ **ผสมกันได้อิสระ** ถ้าใช้ inheritance สร้าง subclass สำหรับทุกชุดค่าผสม
(`EspressoWithMilk`, `EspressoWithMilkAndSugar`, `EspressoWithMilkAndSugarAndCream`, ...)
จำนวน class จะระเบิดแบบ exponential ตามจำนวน topping — 3 topping ก็มีชุดค่าผสมได้ถึง 8 แบบ
แล้ว **Decorator Pattern** แก้ปัญหานี้โดยให้แต่ละ "ตัวเสริม" เป็น class ที่ **ห่อ (wrap)**
object เดิมไว้ข้างใน แล้วยังคง implement interface เดียวกันกับ object ที่มันห่ออยู่

```
┌───────────┐
│  Beverage  │ (Component interface)
├───────────┤
│+description()*│
│+cost()*    │
└───────────┘
      △
   ┌──┴────────────┬─────────────────────┐
┌──────────┐  ┌──────────────────┐
│ Espresso │  │ BeverageDecorator │ (ห่อ Beverage อีกตัวไว้ข้างใน)
└──────────┘  ├──────────────────┤
                │-wrapped_: unique_ptr<Beverage>│
                └──────────────────┘
                        △
          ┌─────────────┼─────────────┐
    ┌───────────┐ ┌──────────────┐ ┌────────────────────┐
    │MilkDecorator│ │SugarDecorator│ │WhippedCreamDecorator│
    └───────────┘ └──────────────┘ └────────────────────┘
    (ห่อซ้อนกันได้ไม่จำกัด: WhippedCream(Sugar(Milk(Espresso))))
```

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <utility>

// ---------- Component ----------
class Beverage {
public:
    virtual ~Beverage() = default;
    virtual std::string description() const = 0;
    virtual double cost() const = 0;
};

// ---------- Concrete Component ----------
class Espresso : public Beverage {
public:
    std::string description() const override { return "เอสเปรสโซ"; }
    double cost() const override { return 60.0; }
};

// ---------- Decorator ฐาน: ห่อ Beverage อีกตัวไว้ข้างใน แล้วยังเป็น Beverage ด้วยตัวเอง ----------
class BeverageDecorator : public Beverage {
public:
    explicit BeverageDecorator(std::unique_ptr<Beverage> wrapped)
        : wrapped_(std::move(wrapped)) {}

protected:
    std::unique_ptr<Beverage> wrapped_;
};

class MilkDecorator : public BeverageDecorator {
public:
    using BeverageDecorator::BeverageDecorator;

    std::string description() const override {
        return wrapped_->description() + " + นม";
    }
    double cost() const override { return wrapped_->cost() + 10.0; }
};

class SugarDecorator : public BeverageDecorator {
public:
    using BeverageDecorator::BeverageDecorator;

    std::string description() const override {
        return wrapped_->description() + " + น้ำตาล";
    }
    double cost() const override { return wrapped_->cost() + 5.0; }
};

class WhippedCreamDecorator : public BeverageDecorator {
public:
    using BeverageDecorator::BeverageDecorator;

    std::string description() const override {
        return wrapped_->description() + " + วิปครีม";
    }
    double cost() const override { return wrapped_->cost() + 15.0; }
};

void printOrder(const Beverage& b) {
    std::cout << b.description() << " ราคา " << b.cost() << " บาท\n";
}

int main() {
    // ประกอบเครื่องดื่มโดยการ "ห่อ" ทับกันเรื่อยๆ ที่ runtime แทนการสร้าง subclass
    // แยกทุกชุดค่าผสม (EspressoWithMilk, EspressoWithMilkAndSugar, ...) ซึ่งจะระเบิดจำนวน class
    std::unique_ptr<Beverage> plain = std::make_unique<Espresso>();
    printOrder(*plain);

    std::unique_ptr<Beverage> withMilk =
        std::make_unique<MilkDecorator>(std::make_unique<Espresso>());
    printOrder(*withMilk);

    std::unique_ptr<Beverage> fullyLoaded = std::make_unique<WhippedCreamDecorator>(
        std::make_unique<SugarDecorator>(
            std::make_unique<MilkDecorator>(std::make_unique<Espresso>())));
    printOrder(*fullyLoaded);
}
```

ผลลัพธ์:

```
เอสเปรสโซ ราคา 60 บาท
เอสเปรสโซ + นม ราคา 70 บาท
เอสเปรสโซ + นม + น้ำตาล + วิปครีม ราคา 90 บาท
```

สังเกตบรรทัด `using BeverageDecorator::BeverageDecorator;` — นี่คือ **inheriting
constructor** (ฟีเจอร์ตั้งแต่ C++11) ที่ดึง constructor ของ base class มาใช้ใน derived class
โดยไม่ต้องเขียน constructor ซ้ำเอง (ทบทวนได้จาก Part 69) ทำให้ `MilkDecorator`,
`SugarDecorator`, `WhippedCreamDecorator` แต่ละตัวไม่ต้องเขียน constructor ที่รับ
`unique_ptr<Beverage>` ซ้ำๆ กันเอง

ข้อสังเกตสำคัญ: Decorator กับ Builder (Part 97) ดูคล้ายกันตรงที่ใช้ chaining/wrapping
แต่ **จุดประสงค์ต่างกันโดยสิ้นเชิง** — Builder ประกอบ "ค่า config" ของ object เดียวก่อนสร้าง
เสร็จสมบูรณ์ครั้งเดียว ส่วน Decorator ห่อ **object ที่สมบูรณ์แล้ว** ซ้อนกันเพื่อเพิ่มพฤติกรรม
และแต่ละชั้นของ Decorator ยังคงเป็น object ที่ใช้งานได้สมบูรณ์ในตัวเองตลอดเวลา

---

## 98.3 Facade Pattern (Step 779)

### ปัญหาที่ Facade แก้: Subsystem ที่ซับซ้อนเกินไปสำหรับผู้ใช้ทั่วไป

ระบบใหญ่มักประกอบด้วย subsystem หลายตัวที่แต่ละตัวมี API ละเอียดของตัวเอง (เช่น ระบบ
โฮมเธียเตอร์ที่มี Amplifier, DVD Player, Projector แยกกัน) การให้ client เรียก API ของทุก
subsystem โดยตรงทำให้โค้ด client ซับซ้อนและผูกติดกับรายละเอียดภายในมากเกินไป
**Facade Pattern** แก้ปัญหานี้โดยสร้าง class หน้าด่านตัวเดียวที่ **ซ่อนความซับซ้อนทั้งหมด** ไว้
ข้างหลัง แล้วเปิด method ง่ายๆ ไม่กี่ตัวให้ client เรียกใช้

```
┌──────────────────┐
│HomeTheaterFacade  │ (interface ง่ายๆ)
├──────────────────┤
│+watchMovie(title) │───┬──▶ Projector.on(), setWideScreenMode()
│+endMovie()        │   ├──▶ Amplifier.on(), setVolume()
└──────────────────┘   └──▶ DvdPlayer.on(), play()
        ▲
        │ client เรียกแค่ 2 method
      main()
```

```cpp
#include <iostream>
#include <string>

// ---------- Subsystems ที่ซับซ้อน มี API ของตัวเองแยกกัน ----------
class Amplifier {
public:
    void on() { std::cout << "  Amplifier: เปิดเครื่อง\n"; }
    void setVolume(int level) { std::cout << "  Amplifier: ตั้งระดับเสียง " << level << "\n"; }
    void off() { std::cout << "  Amplifier: ปิดเครื่อง\n"; }
};

class DvdPlayer {
public:
    void on() { std::cout << "  DvdPlayer: เปิดเครื่อง\n"; }
    void play(const std::string& movie) { std::cout << "  DvdPlayer: เล่น \"" << movie << "\"\n"; }
    void stop() { std::cout << "  DvdPlayer: หยุดเล่น\n"; }
    void off() { std::cout << "  DvdPlayer: ปิดเครื่อง\n"; }
};

class Projector {
public:
    void on() { std::cout << "  Projector: เปิดเครื่อง\n"; }
    void setWideScreenMode() { std::cout << "  Projector: ตั้งโหมดจอกว้าง\n"; }
    void off() { std::cout << "  Projector: ปิดเครื่อง\n"; }
};

// ---------- Facade: ให้ interface เดียวที่ง่าย ครอบ subsystem ทั้งหมดไว้ ----------
class HomeTheaterFacade {
public:
    HomeTheaterFacade(Amplifier& amp, DvdPlayer& dvd, Projector& projector)
        : amp_(amp), dvd_(dvd), projector_(projector) {}

    void watchMovie(const std::string& movie) {
        std::cout << "กำลังเตรียมดูหนัง...\n";
        projector_.on();
        projector_.setWideScreenMode();
        amp_.on();
        amp_.setVolume(7);
        dvd_.on();
        dvd_.play(movie);
    }

    void endMovie() {
        std::cout << "กำลังปิดระบบโฮมเธียเตอร์...\n";
        dvd_.stop();
        dvd_.off();
        amp_.off();
        projector_.off();
    }

private:
    Amplifier& amp_;
    DvdPlayer& dvd_;
    Projector& projector_;
};

int main() {
    Amplifier amp;
    DvdPlayer dvd;
    Projector projector;
    HomeTheaterFacade theater(amp, dvd, projector);

    // ผู้ใช้เรียกแค่ 2 คำสั่ง โดยไม่ต้องรู้เลยว่าเบื้องหลังต้องสั่ง subsystem กี่ตัว
    theater.watchMovie("Inception");
    theater.endMovie();
}
```

ผลลัพธ์:

```
กำลังเตรียมดูหนัง...
  Projector: เปิดเครื่อง
  Projector: ตั้งโหมดจอกว้าง
  Amplifier: เปิดเครื่อง
  Amplifier: ตั้งระดับเสียง 7
  DvdPlayer: เปิดเครื่อง
  DvdPlayer: เล่น "Inception"
กำลังปิดระบบโฮมเธียเตอร์...
  DvdPlayer: หยุดเล่น
  DvdPlayer: ปิดเครื่อง
  Amplifier: ปิดเครื่อง
  Projector: ปิดเครื่อง
```

**ข้อสำคัญที่มือใหม่มักเข้าใจผิด**: Facade **ไม่ได้ปิดกั้น** การเข้าถึง subsystem โดยตรง —
ถ้า client ต้องการควบคุม `Amplifier` แบบละเอียดกว่าที่ `HomeTheaterFacade` มีให้ ก็ยังเรียก
`amp.setVolume(10)` ตรงๆ ได้เสมอ (สังเกตว่า `amp_`, `dvd_`, `projector_` ในตัวอย่างเป็น
reference ไปยัง object ภายนอก ไม่ได้ถูกซ่อนสมบูรณ์) Facade เป็นแค่ **"ทางลัดที่สะดวก"** สำหรับ
กรณีใช้งานทั่วไป 80% ไม่ใช่กำแพงที่ปิดกั้นการเข้าถึงรายละเอียด 20% ที่เหลือ

---

## 98.4 Composite Pattern (Step 780)

### ปัญหาที่ Composite แก้: ต้นไม้ที่มีทั้ง "หน่วยเดี่ยว" และ "กลุ่ม"

ระบบไฟล์คือตัวอย่างคลาสสิกที่สุดของปัญหานี้: โฟลเดอร์หนึ่งอันมีทั้งไฟล์เดี่ยวและโฟลเดอร์ย่อย
ปนกันอยู่ และโฟลเดอร์ย่อยก็มีไฟล์/โฟลเดอร์ย่อยอีกชั้นซ้อนไปเรื่อยๆ ถ้าเราต้องการฟังก์ชัน
"คำนวณขนาดรวม" หรือ "แสดงรายการทั้งหมด" โค้ดที่ต้องแยกกรณี "ถ้าเป็นไฟล์ทำแบบนี้ ถ้าเป็น
โฟลเดอร์ทำอีกแบบ" จะซับซ้อนขึ้นเรื่อยๆ ตามความลึกของ tree **Composite Pattern** แก้ปัญหานี้
โดยให้ทั้ง "หน่วยเดี่ยว" (Leaf) และ "กลุ่ม" (Composite) implement **interface เดียวกัน** ทำให้
client เขียนโค้ดปฏิบัติกับทั้งคู่แบบเดียวกันได้ทั้งหมด โดยไม่ต้องสนใจว่ากำลังคุยกับไฟล์เดี่ยว
หรือทั้งโฟลเดอร์

```
┌─────────────────────┐
│  FileSystemComponent  │ (Component interface)
├─────────────────────┤
│+sizeBytes()*          │
│+print()*              │
└─────────────────────┘
        △
   ┌────┴─────────────┐
┌────────┐      ┌───────────────────────┐
│  File   │      │      Directory          │ (Composite)
│ (Leaf)  │      ├───────────────────────┤
└────────┘      │-children: vector<unique_ptr<FileSystemComponent>>│
                 │+add(child)              │
                 └──────────┬────────────┘
                             │ เก็บได้ทั้ง File และ Directory ตัวอื่น (ซ้อนกันได้)
                             ▼
                   (เรียก sizeBytes()/print() ของลูกแบบ recursive)
```

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

// ---------- Component: interface ร่วมของทั้ง "ไฟล์เดี่ยว" และ "โฟลเดอร์ที่มีลูก" ----------
class FileSystemComponent {
public:
    explicit FileSystemComponent(std::string name) : name_(std::move(name)) {}
    virtual ~FileSystemComponent() = default;

    virtual long sizeBytes() const = 0;
    virtual void print(int indent) const = 0;
    void print() const { print(0); }   // เรียกแบบไม่ระบุ indent ได้ (ค่าเริ่มต้น = 0)

protected:
    std::string indentStr(int indent) const { return std::string(static_cast<std::size_t>(indent) * 2, ' '); }
    std::string name_;
};

// ---------- Leaf: ไฟล์เดี่ยว ไม่มีลูก ----------
class File : public FileSystemComponent {
public:
    using FileSystemComponent::print;   // ดึง print() แบบไม่มี argument กลับมาให้มองเห็น
                                         // (มิเช่นนั้นการประกาศ print(int) ทับชื่อเดิมไปหมด)
    File(std::string name, long size) : FileSystemComponent(std::move(name)), size_(size) {}

    long sizeBytes() const override { return size_; }

    void print(int indent) const override {
        std::cout << indentStr(indent) << "- " << name_ << " (" << size_ << " bytes)\n";
    }

private:
    long size_;
};

// ---------- Composite: โฟลเดอร์ ที่เก็บ FileSystemComponent ลูกได้ทั้งไฟล์และโฟลเดอร์ย่อย ----------
class Directory : public FileSystemComponent {
public:
    using FileSystemComponent::print;   // เช่นเดียวกับ File: ดึง print() แบบไม่มี argument กลับมา
    explicit Directory(std::string name) : FileSystemComponent(std::move(name)) {}

    void add(std::unique_ptr<FileSystemComponent> child) {
        children_.push_back(std::move(child));
    }

    long sizeBytes() const override {
        long total = 0;
        for (const auto& child : children_) {
            total += child->sizeBytes();   // เรียกซ้ำ (recursive) โดยไม่สนว่าลูกเป็นไฟล์หรือโฟลเดอร์
        }
        return total;
    }

    void print(int indent) const override {
        std::cout << indentStr(indent) << "+ " << name_ << "/ (" << sizeBytes() << " bytes รวม)\n";
        for (const auto& child : children_) {
            child->print(indent + 1);
        }
    }

private:
    std::vector<std::unique_ptr<FileSystemComponent>> children_;
};

int main() {
    auto root = std::make_unique<Directory>("project");
    root->add(std::make_unique<File>("README.md", 1200));

    auto srcDir = std::make_unique<Directory>("src");
    srcDir->add(std::make_unique<File>("main.cpp", 3400));
    srcDir->add(std::make_unique<File>("utils.cpp", 2100));

    auto testDir = std::make_unique<Directory>("tests");
    testDir->add(std::make_unique<File>("test_main.cpp", 1800));

    srcDir->add(std::move(testDir));   // โฟลเดอร์ซ้อนโฟลเดอร์ได้ เพราะ Directory ก็เป็น
                                        // FileSystemComponent เหมือนกัน
    root->add(std::move(srcDir));

    root->print();   // client เรียก print()/sizeBytes() เดียว โดยไม่สนโครงสร้างข้างในเลย
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 composite.cpp -o composite
./composite
```

ผลลัพธ์:

```
+ project/ (8500 bytes รวม)
  - README.md (1200 bytes)
  + src/ (7300 bytes รวม)
    - main.cpp (3400 bytes)
    - utils.cpp (2100 bytes)
    + tests/ (1800 bytes รวม)
      - test_main.cpp (1800 bytes)
```

> **หมายเหตุเรื่อง `using FileSystemComponent::print;`**: เมื่อ derived class ประกาศฟังก์ชัน
> ชื่อเดียวกับ base class (ในที่นี้คือ `print`) ไม่ว่าจะคนละ signature ก็ตาม กฎ **name hiding**
> ของ C++ จะทำให้ overload ทั้งหมดของชื่อนั้นใน base class ถูก "บัง" ไปหมดเมื่อเรียกผ่าน
> derived type โดยตรง การเขียน `using Base::functionName;` คือวิธีมาตรฐานในการดึง overload
> ที่ถูกบังนั้นกลับมาให้มองเห็นอีกครั้ง (ทบทวนกฎ name hiding นี้ได้จาก Module D)

---

## 98.5 Behavioral Pattern คืออะไร + Observer Pattern (Step 781)

### Behavioral Pattern คืออะไร

**Behavioral Pattern** เน้นที่ **"การกระจายความรับผิดชอบและการสื่อสารระหว่าง object"**
ต่างจาก Creational (สร้าง) และ Structural (ประกอบ) — คำถามหลักของหมวดนี้คือ "เมื่อ object
A ต้องทำงานร่วมกับ object B จะให้ทั้งคู่สื่อสารกันอย่างไรโดยไม่ผูกติดกันแน่นเกินไป" หนังสือ
GoF มี Behavioral Pattern ทั้งหมด 11 แบบ บทนี้จะเจาะลึก 4 แบบที่สำคัญที่สุดสำหรับ C++ และ
สำหรับ Web Development ที่กำลังจะเรียนใน Module I: **Observer, Strategy, Command, Iterator**

### ปัญหาที่ Observer Pattern แก้

ระบบจำนวนมากต้องการให้ "เมื่อ A เปลี่ยนแปลง ให้ B, C, D รู้ทันที" โดยที่ A **ไม่ควรรู้จัก B,
C, D โดยตรง** (เพราะจำนวนและชนิดของผู้รับการแจ้งเตือนอาจเปลี่ยนได้ตลอดเวลา) เช่น ระบบราคา
หุ้นที่ต้องอัปเดตหลายหน้าจอพร้อมกัน หรือปุ่มกดใน GUI ที่ต้องแจ้ง handler หลายตัว **Observer
Pattern** แก้ปัญหานี้ด้วยความสัมพันธ์แบบ **publish-subscribe**: `Subject` เก็บรายชื่อ
`Observer` ที่ subscribe ไว้ แล้วเรียก `notify()` ไปยังทุกตัวเมื่อมีการเปลี่ยนแปลง

```
┌────────────┐  notify   ┌──────────────────┐
│StockTicker  │──────────▷│  StockObserver    │ (interface)
│ (Subject)   │           ├──────────────────┤
├────────────┤           │+onPriceChanged()* │
│-observers_:  │           └──────────────────┘
│ vector<weak_ptr<StockObserver>>│      △
│+subscribe(obs)│               ┌──────┴──────┐
│+setPrice(...)│         ┌──────────────┐ ┌──────────────┐
└────────────┘         │PriceDisplay(A)│ │PriceDisplay(B)│
                         └──────────────┘ └──────────────┘
```

```cpp
#include <algorithm>
#include <iostream>
#include <map>
#include <memory>
#include <string>
#include <vector>

// ---------- Observer interface ----------
class StockObserver {
public:
    virtual ~StockObserver() = default;
    virtual void onPriceChanged(const std::string& symbol, double price) = 0;
};

// ---------- Subject ----------
// เก็บผู้สังเกตการณ์ด้วย weak_ptr แทน raw pointer หรือ shared_ptr ตรงๆ:
// - raw pointer: ถ้า observer ถูกลบไปแล้วแต่ลืม unregister -> dangling pointer -> UB ทันที
// - shared_ptr ตรงๆ ใน subject: subject จะถือ ownership ไว้ ทำให้ observer ไม่มีวันถูกทำลาย
//   ตราบใดที่ subject ยังไม่ unregister ให้ -> วนไปเป็นปัญหา "memory ที่ไม่ควรมีชีวิตอยู่ต่อ"
// - weak_ptr: subject "สังเกต" ได้แต่ไม่ถือ ownership เมื่อ observer ตายไปแล้ว weak_ptr.lock()
//   จะคืน nullptr ให้เอง ปลอดภัยกว่าทั้งสองแบบข้างต้น
class StockTicker {
public:
    void subscribe(const std::shared_ptr<StockObserver>& observer) {
        observers_.push_back(observer);
    }

    void setPrice(const std::string& symbol, double price) {
        prices_[symbol] = price;
        notify(symbol, price);
    }

private:
    void notify(const std::string& symbol, double price) {
        // ทำความสะอาด weak_ptr ที่ observer ตายไปแล้วออกไปด้วย (erase-remove idiom)
        observers_.erase(
            std::remove_if(observers_.begin(), observers_.end(),
                            [](const std::weak_ptr<StockObserver>& w) { return w.expired(); }),
            observers_.end());

        for (const auto& weakObs : observers_) {
            if (auto obs = weakObs.lock()) {   // lock() คืน shared_ptr ชั่วคราวถ้ายังไม่ตาย
                obs->onPriceChanged(symbol, price);
            }
        }
    }

    std::vector<std::weak_ptr<StockObserver>> observers_;
    std::map<std::string, double> prices_;
};

class PriceDisplay : public StockObserver {
public:
    explicit PriceDisplay(std::string name) : name_(std::move(name)) {}

    void onPriceChanged(const std::string& symbol, double price) override {
        std::cout << "[" << name_ << "] " << symbol << " = " << price << " บาท\n";
    }

private:
    std::string name_;
};

int main() {
    StockTicker ticker;

    auto mobileDisplay = std::make_shared<PriceDisplay>("Mobile App");
    ticker.subscribe(mobileDisplay);

    {
        auto webDisplay = std::make_shared<PriceDisplay>("Web Dashboard");
        ticker.subscribe(webDisplay);

        std::cout << "-- ตอนที่ webDisplay ยังมีชีวิตอยู่ --\n";
        ticker.setPrice("PTT", 35.5);
    }   // webDisplay หลุด scope ตรงนี้ -> shared_ptr ตัวสุดท้ายถูกทำลาย -> object ถูก destroy จริง

    std::cout << "-- หลัง webDisplay ถูกทำลายไปแล้ว --\n";
    ticker.setPrice("PTT", 36.0);   // แจ้งเตือนแค่ mobileDisplay เพราะ weak_ptr ของ webDisplay
                                     // expired แล้ว จึงถูกกรองทิ้งอัตโนมัติ ไม่มีการเข้าถึง
                                     // memory ที่ถูก free ไปแล้ว (ไม่เกิด UB)
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 observer.cpp -o observer
./observer
```

ผลลัพธ์:

```
-- ตอนที่ webDisplay ยังมีชีวิตอยู่ --
[Mobile App] PTT = 35.5 บาท
[Web Dashboard] PTT = 35.5 บาท
-- หลัง webDisplay ถูกทำลายไปแล้ว --
[Mobile App] PTT = 36 บาท
```

สังเกตว่าหลัง `webDisplay` หลุด scope และถูกทำลายไปแล้ว การเรียก `setPrice()` ครั้งที่สอง
**ไม่ crash และไม่แจ้งเตือน `webDisplay` อีกเลย** เพราะ `weak_ptr::expired()` ตรวจพบว่า
object ต้นทางตายไปแล้ว และโค้ดใน `notify()` กรอง entry นั้นทิ้งไปโดยอัตโนมัติก่อนจะพยายาม
เรียกใช้งานมัน — นี่คือเหตุผลที่ตัวอย่าง Observer สมัยใหม่ควรใช้ `weak_ptr` แทน raw pointer
เสมอเมื่อ subject และ observer มีอายุ (lifetime) ที่ควบคุมแยกจากกัน หัวข้อนี้จะสำคัญมากใน
**Module I** เมื่อเราสร้างระบบ event-driven สำหรับ Web Server ที่มี handler จำนวนมาก
subscribe/unsubscribe ตลอดเวลา

---

## 98.6 Strategy Pattern (Step 782)

### ปัญหาที่ Strategy แก้: Algorithm ที่ต้องสลับได้ที่ Runtime

เมื่อมี algorithm หลายแบบสำหรับงานเดียวกัน (เช่น วิธีชำระเงินหลายแบบ, วิธี sort หลายแบบ,
วิธี compress หลายแบบ) และต้องการ **สลับใช้แบบไหนก็ได้ที่ runtime** โดยไม่แก้โค้ดของฝั่งที่
เรียกใช้ **Strategy Pattern** แก้ปัญหานี้โดยห่อแต่ละ algorithm เป็น object แยกกันที่มี
interface เดียวกัน แล้วให้ผู้ใช้ "เสียบ" strategy ตัวไหนก็ได้เข้าไป

```
┌──────────────┐  ใช้ผ่าน   ┌──────────────────┐
│ ShoppingCart  │──────────▷│ PaymentStrategy    │ (interface)
├──────────────┤           ├──────────────────┤
│-strategy_: unique_ptr│    │+pay(amount)*      │
│+checkout(amount)│         └──────────────────┘
└──────────────┘                    △
                              ┌─────┴─────┐
                    ┌───────────────┐ ┌──────────────┐
                    │CreditCardStrategy│ │PayPalStrategy│
                    └───────────────┘ └──────────────┘
```

```cpp
#include <functional>
#include <iostream>
#include <memory>
#include <string>
#include <utility>

// ---------- Strategy interface (แบบคลาสสิก: ใช้ class hierarchy) ----------
class PaymentStrategy {
public:
    virtual ~PaymentStrategy() = default;
    virtual void pay(double amount) const = 0;
};

class CreditCardStrategy : public PaymentStrategy {
public:
    explicit CreditCardStrategy(std::string cardNumber) : cardNumber_(std::move(cardNumber)) {}

    void pay(double amount) const override {
        std::cout << "ชำระ " << amount << " บาท ด้วยบัตรเครดิตเลขลงท้าย "
                  << cardNumber_.substr(cardNumber_.size() - 4) << '\n';
    }

private:
    std::string cardNumber_;
};

class PayPalStrategy : public PaymentStrategy {
public:
    explicit PayPalStrategy(std::string email) : email_(std::move(email)) {}

    void pay(double amount) const override {
        std::cout << "ชำระ " << amount << " บาท ผ่าน PayPal (" << email_ << ")\n";
    }

private:
    std::string email_;
};

class ShoppingCart {
public:
    void setPaymentStrategy(std::unique_ptr<PaymentStrategy> strategy) {
        strategy_ = std::move(strategy);
    }

    void checkout(double amount) const {
        if (!strategy_) {
            std::cout << "ยังไม่ได้เลือกวิธีชำระเงิน\n";
            return;
        }
        strategy_->pay(amount);   // ShoppingCart ไม่รู้เลยว่ากำลังใช้วิธีจ่ายแบบไหนอยู่
    }

private:
    std::unique_ptr<PaymentStrategy> strategy_;
};

// ---------- Strategy แบบ modern C++: ใช้ std::function + lambda แทน class hierarchy ----------
// เหมาะกับกรณีที่ strategy เป็นแค่ "พฤติกรรมสั้นๆ" ไม่มี state ซับซ้อนของตัวเอง
// (เชื่อมโยงกับ lambda/std::function จาก Part 64)
void checkoutWithFunction(double amount, const std::function<void(double)>& payFn) {
    payFn(amount);
}

int main() {
    ShoppingCart cart;

    cart.setPaymentStrategy(std::make_unique<CreditCardStrategy>("4111111111111234"));
    cart.checkout(1500.0);

    cart.setPaymentStrategy(std::make_unique<PayPalStrategy>("nan@example.com"));
    cart.checkout(750.0);

    std::cout << "-- เวอร์ชัน std::function + lambda --\n";
    checkoutWithFunction(300.0, [](double amount) {
        std::cout << "ชำระ " << amount << " บาท ด้วยเงินสดพร้อมส่วนลด 5%\n";
    });
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 strategy.cpp -o strategy
./strategy
```

ผลลัพธ์:

```
ชำระ 1500 บาท ด้วยบัตรเครดิตเลขลงท้าย 1234
ชำระ 750 บาท ผ่าน PayPal (nan@example.com)
-- เวอร์ชัน std::function + lambda --
ชำระ 300 บาท ด้วยเงินสดพร้อมส่วนลด 5%
```

### เมื่อไหร่ใช้ class hierarchy เมื่อไหร่ใช้ std::function

| ใช้ **class hierarchy** (`PaymentStrategy`) เมื่อ | ใช้ **`std::function` + lambda** เมื่อ |
|---|---|
| Strategy มี state ภายในของตัวเอง (เช่น `cardNumber_`) | Strategy เป็นแค่ฟังก์ชันสั้นๆ ไม่มี state |
| ต้องการ polymorphism แบบเต็มรูปแบบ (หลาย virtual method) | ต้องการแค่ "พฤติกรรมเดียว" ที่เปลี่ยนได้ |
| อยากบังคับ interface ชัดเจนผ่าน pure virtual function | อยากให้ผู้เรียกเขียน lambda inline ได้สะดวก |

Modern C++ มักเลือกใช้ `std::function`/lambda สำหรับ Strategy ที่เรียบง่าย เพราะลดจำนวน
class ที่ต้องเขียน แต่เมื่อ strategy ซับซ้อนขึ้น (มี state, มีหลาย method ที่เกี่ยวข้องกัน) การใช้
class hierarchy แบบคลาสสิกยังคงเป็นทางเลือกที่ชัดเจนและทดสอบง่ายกว่า

---

## 98.7 Command Pattern (Step 783)

### ปัญหาที่ Command Pattern แก้: ต้องการ "คำสั่ง" ที่เป็น Object เก็บไว้ ส่งต่อได้ Undo ได้

บางระบบต้องการมากกว่าแค่ "เรียกฟังก์ชันแล้วจบ" เช่น ต้องการ **เก็บประวัติคำสั่งไว้ทำ undo**
(โปรแกรมแก้ไขข้อความ), **จัดคิวคำสั่งไว้ทำทีหลัง** (task queue), หรือ **ส่งคำสั่งข้ามระบบ**
(remote control) **Command Pattern** แก้ปัญหานี้โดยห่อ "การกระทำหนึ่งครั้ง" ให้เป็น **object**
ที่มี method `execute()` (และมักมี `undo()` คู่กัน) แทนที่จะเรียกฟังก์ชันตรงๆ

```
┌───────────────┐  เก็บประวัติ   ┌────────────┐
│ RemoteControl  │──────────────▷│  Command    │ (interface)
│  (Invoker)     │  vector<unique_ptr<Command>>│├────────────┤
├───────────────┤                │+execute()* │
│+pressButton(cmd)│                │+undo()*    │
│+pressUndo()    │                └────────────┘
└───────────────┘                       △
                                  ┌──────┴──────┐
                          ┌───────────────┐ ┌────────────────┐
                          │LightOnCommand  │ │LightOffCommand  │
                          └───────┬───────┘ └────────┬───────┘
                                  │ ควบคุม              │
                                  ▼                    ▼
                              ┌────────────────────┐
                              │      Light (Receiver)│
                              └────────────────────┘
```

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

// ---------- Receiver: ตัวที่ทำงานจริง ----------
class Light {
public:
    explicit Light(std::string room) : room_(std::move(room)) {}

    void on() {
        isOn_ = true;
        std::cout << "ไฟห้อง " << room_ << ": เปิด\n";
    }

    void off() {
        isOn_ = false;
        std::cout << "ไฟห้อง " << room_ << ": ปิด\n";
    }

private:
    std::string room_;
    bool isOn_ = false;
};

// ---------- Command interface ----------
class Command {
public:
    virtual ~Command() = default;
    virtual void execute() = 0;
    virtual void undo() = 0;
};

class LightOnCommand : public Command {
public:
    explicit LightOnCommand(Light& light) : light_(light) {}
    void execute() override { light_.on(); }
    void undo() override { light_.off(); }

private:
    Light& light_;
};

class LightOffCommand : public Command {
public:
    explicit LightOffCommand(Light& light) : light_(light) {}
    void execute() override { light_.off(); }
    void undo() override { light_.on(); }

private:
    Light& light_;
};

// ---------- Invoker: เก็บประวัติคำสั่งไว้ทำ undo ได้ โดยไม่ต้องรู้จัก Light เลย ----------
class RemoteControl {
public:
    void pressButton(std::unique_ptr<Command> command) {
        command->execute();
        history_.push_back(std::move(command));
    }

    void pressUndo() {
        if (history_.empty()) {
            std::cout << "ไม่มีคำสั่งให้ undo แล้ว\n";
            return;
        }
        history_.back()->undo();
        history_.pop_back();
    }

private:
    std::vector<std::unique_ptr<Command>> history_;
};

int main() {
    Light livingRoom("นั่งเล่น");
    RemoteControl remote;

    remote.pressButton(std::make_unique<LightOnCommand>(livingRoom));
    remote.pressButton(std::make_unique<LightOffCommand>(livingRoom));

    std::cout << "-- กด undo 2 ครั้ง --\n";
    remote.pressUndo();   // undo "ปิดไฟ" -> กลับไปเป็น "เปิดไฟ"
    remote.pressUndo();   // undo "เปิดไฟ" -> กลับไปเป็น "ปิดไฟ"

    remote.pressUndo();   // ไม่มีอะไรให้ undo แล้ว
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 command.cpp -o command
./command
```

ผลลัพธ์:

```
ไฟห้อง นั่งเล่น: เปิด
ไฟห้อง นั่งเล่น: ปิด
-- กด undo 2 ครั้ง --
ไฟห้อง นั่งเล่น: เปิด
ไฟห้อง นั่งเล่น: ปิด
ไม่มีคำสั่งให้ undo แล้ว
```

จุดสำคัญ: `RemoteControl` (Invoker) **ไม่รู้จัก `Light` เลยแม้แต่น้อย** — มันรู้แค่ว่ามี
`Command` ที่ `execute()`/`undo()` ได้ ทำให้ `RemoteControl` เดิมสามารถควบคุมอุปกรณ์ชนิดใหม่
(เช่น พัดลม, ประตูอัตโนมัติ) ได้ทันทีเพียงแค่เขียน `Command` concrete class ใหม่ โดยไม่ต้องแก้
`RemoteControl` เลย — นี่คือ Open/Closed Principle อีกครั้งหนึ่งที่ปรากฏซ้ำในหลาย pattern

---

## 98.8 Iterator Pattern และการปิดท้าย Module H (Step 784)

### ปัญหาที่ Iterator Pattern แก้: เข้าถึงข้อมูลภายใน Container โดยไม่เปิดเผยโครงสร้าง

ใน **Part 62** เราเรียนรู้ STL Iterator (`begin()`, `end()`, `iterator_category` ทั้ง 5 แบบ)
ไปอย่างละเอียดแล้วว่ามันคือกลไกที่ generalize แนวคิด pointer ให้ใช้กับ container ได้ทุกชนิด
สิ่งที่ Part 62 ไม่ได้บอกตรงๆ คือ **สิ่งที่ STL ทำอยู่นั้นคือ GoF Iterator Pattern** ที่ถูก
formalize เป็นมาตรฐานของภาษา — Iterator Pattern บอกว่า: **container ควรมี object แยก
ต่างหาก (iterator) ที่รู้วิธี "เดิน" ผ่านข้อมูลภายใน โดยที่ผู้ใช้ container ไม่ต้องรู้เลยว่า
โครงสร้างข้อมูลภายในเป็นอะไร** (array, linked list, tree, ...)

ตัวอย่างนี้สร้าง container ของเราเอง (`Playlist`) พร้อม iterator ที่เข้ากันได้กับ range-based
for loop และ STL algorithm ทุกประการ เหมือนที่ `std::vector`/`std::list` ทำ:

```
┌────────────┐             ┌───────────────────┐
│  Playlist   │  ประกาศ     │ Playlist::Iterator  │
├────────────┤ ─────────▷ ├───────────────────┤
│-songs_: vector<string>│  │-it_: vector<string>::const_iterator│
│+begin()     │            │+operator*()        │
│+end()       │            │+operator++()       │
└────────────┘            │+operator==/!=()    │
                            └───────────────────┘
        ▲
        │ ใช้กับ range-based for และ std::find/std::distance ได้เหมือน container มาตรฐาน
      main()
```

```cpp
#include <algorithm>
#include <cstddef>
#include <iostream>
#include <iterator>
#include <string>
#include <vector>

// Playlist คือ container "ของเราเอง" ที่ต้องการให้ใช้งานได้เหมือน STL container ทุกประการ:
// range-based for loop, std::find, std::for_each ฯลฯ โดยไม่เปิดเผยโครงสร้างข้อมูลภายใน
// (ในตัวอย่างนี้ใช้ std::vector<std::string> เก็บข้อมูลจริง แต่ผู้ใช้ Playlist ไม่ควรต้องรู้)
class Playlist {
public:
    void addSong(std::string title) { songs_.push_back(std::move(title)); }

    // ---------- GoF Iterator Pattern: สร้าง iterator class ของเราเอง ----------
    class Iterator {
    public:
        // typedef มาตรฐานที่ STL algorithm ต้องใช้ตรวจสอบความสามารถของ iterator
        using iterator_category = std::forward_iterator_tag;
        using value_type = std::string;
        using difference_type = std::ptrdiff_t;
        using pointer = const std::string*;
        using reference = const std::string&;

        explicit Iterator(std::vector<std::string>::const_iterator it) : it_(it) {}

        reference operator*() const { return *it_; }
        pointer operator->() const { return &(*it_); }

        Iterator& operator++() {        // prefix ++
            ++it_;
            return *this;
        }
        Iterator operator++(int) {      // postfix ++
            Iterator tmp = *this;
            ++(*this);
            return tmp;
        }

        bool operator==(const Iterator& other) const { return it_ == other.it_; }
        bool operator!=(const Iterator& other) const { return it_ != other.it_; }

    private:
        std::vector<std::string>::const_iterator it_;
    };

    Iterator begin() const { return Iterator(songs_.cbegin()); }
    Iterator end() const { return Iterator(songs_.cend()); }

private:
    std::vector<std::string> songs_;
};

int main() {
    Playlist playlist;
    playlist.addSong("Bohemian Rhapsody");
    playlist.addSong("Hotel California");
    playlist.addSong("Stairway to Heaven");

    // ใช้ range-based for ได้ทันที เพราะมี begin()/end() ตามข้อกำหนด
    std::cout << "-- เล่นเพลงทั้งหมด --\n";
    for (const std::string& song : playlist) {
        std::cout << "  ▶ " << song << '\n';
    }

    // ใช้กับ STL algorithm ได้เหมือน container มาตรฐานทุกประการ เพราะประกาศ
    // iterator_category ไว้ถูกต้องตาม Forward Iterator requirement
    auto found = std::find(playlist.begin(), playlist.end(), "Hotel California");
    if (found != playlist.end()) {
        std::cout << "พบเพลง: " << *found << '\n';
    }

    auto count = std::distance(playlist.begin(), playlist.end());
    std::cout << "จำนวนเพลงทั้งหมด: " << count << '\n';
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 iterator_pattern.cpp -o iterator_pattern
./iterator_pattern
```

ผลลัพธ์:

```
-- เล่นเพลงทั้งหมด --
  ▶ Bohemian Rhapsody
  ▶ Hotel California
  ▶ Stairway to Heaven
พบเพลง: Hotel California
จำนวนเพลงทั้งหมด: 3
```

ทวนจาก Part 62: การประกาศ `iterator_category`, `value_type`, `difference_type`, `pointer`,
`reference` ให้ครบคือสิ่งที่ทำให้ `std::iterator_traits` (และ STL algorithm ทุกตัวที่พึ่งพามัน
เช่น `std::find`, `std::distance`) รู้จัก `Playlist::Iterator` ว่าเป็น **Forward Iterator** ที่
ใช้งานได้ — นี่คือหลักฐานที่ชัดเจนที่สุดว่า **STL ทั้งหมดสร้างอยู่บนหลักการของ GoF Iterator
Pattern** เพียงแต่ทำให้เป็นระบบมาตรฐานที่ generic กว่าตัวอย่าง GoF ดั้งเดิมมาก

### สรุปเปรียบเทียบ Structural และ Behavioral Pattern ทั้ง 8 แบบ

| Pattern | หมวด | แก้ปัญหาอะไร | ใช้เมื่อไหร่ |
|---|---|---|---|
| **Adapter** | Structural | Interface เก่ากับใหม่ไม่ตรงกัน | ต้อง integrate library/ระบบเก่าที่แก้ไม่ได้ |
| **Decorator** | Structural | Subclass explosion จากการผสม feature | เพิ่มพฤติกรรมแบบ dynamic ที่ runtime |
| **Facade** | Structural | Subsystem ซับซ้อนเกินไปสำหรับผู้ใช้ทั่วไป | ต้องการ API ง่ายๆ ครอบระบบที่ซับซ้อน |
| **Composite** | Structural | ต้นไม้ที่มีทั้งหน่วยเดี่ยวและกลุ่ม | โครงสร้างแบบ recursive เช่น ระบบไฟล์, DOM tree |
| **Observer** | Behavioral | ต้องแจ้งเตือนหลาย object เมื่อมีการเปลี่ยนแปลง | Event system, GUI, real-time update |
| **Strategy** | Behavioral | Algorithm ต้องสลับได้ที่ runtime | มี algorithm หลายแบบสำหรับงานเดียวกัน |
| **Command** | Behavioral | ต้องการคำสั่งที่เป็น object เก็บ/ส่งต่อ/undo ได้ | Undo-redo, task queue, remote control |
| **Iterator** | Behavioral | เข้าถึงข้อมูลภายใน container โดยไม่เปิดเผยโครงสร้าง | Container ที่ต้องใช้กับ range-based for/STL algorithm |

---

## อย่า Over-Engineer: Pattern คือเครื่องมือ ไม่ใช่เป้าหมาย

หลังเรียนจบ 9 pattern ใน 2 Part นี้ กับดักที่พบบ่อยที่สุดของผู้เรียนใหม่ (และแม้แต่วิศวกรที่
มีประสบการณ์บางคน) คือ **"อยากใช้ pattern ให้ครบทุกตัวในโปรเจกต์เดียว"** เพื่อแสดงว่ารู้จัก
pattern เยอะ นี่คือความผิดพลาดที่เรียกว่า **Over-Engineering** และเป็นปัญหาที่ร้ายแรงพอๆ กับ
การไม่ใช้ pattern เลยทั้งที่ควรใช้

### ตัวอย่างการ Over-Engineer ด้วย Strategy Pattern

```cpp
#include <iostream>

// ตัวอย่าง "Over-engineering": ใช้ Strategy Pattern เต็มรูปแบบกับปัญหาที่มีทางเลือกแค่ 2 ทาง
// และไม่มีทีท่าว่าจะเพิ่มขึ้นอีกเลย -- เพิ่ม class, virtual function, unique_ptr
// โดยไม่ได้อะไรเพิ่มขึ้นมาเลยเมื่อเทียบกับ if/else ธรรมดา
class DiscountStrategy {
public:
    virtual ~DiscountStrategy() = default;
    virtual double apply(double price) const = 0;
};

class VipDiscount : public DiscountStrategy {
public:
    double apply(double price) const override { return price * 0.8; }
};

class RegularDiscount : public DiscountStrategy {
public:
    double apply(double price) const override { return price; }
};

double checkoutOverEngineered(double price, const DiscountStrategy& strategy) {
    return strategy.apply(price);
}

// ทางเลือกที่ "พอดี" กับปัญหาจริง: เงื่อนไขเดียว อ่านง่ายกว่า แก้ง่ายกว่า ไม่ต้องมี
// virtual dispatch ที่ไม่จำเป็น และไม่ต้องเปิดไฟล์ใหม่เพิ่มทุกครั้งที่มีเงื่อนไขใหม่นิดเดียว
double checkoutSimple(double price, bool isVip) {
    return isVip ? price * 0.8 : price;
}

int main() {
    VipDiscount vip;
    RegularDiscount regular;

    std::cout << "แบบ over-engineered (VIP): " << checkoutOverEngineered(1000.0, vip) << '\n';
    std::cout << "แบบ over-engineered (ปกติ): " << checkoutOverEngineered(1000.0, regular) << '\n';

    std::cout << "แบบเรียบง่าย (VIP): " << checkoutSimple(1000.0, true) << '\n';
    std::cout << "แบบเรียบง่าย (ปกติ): " << checkoutSimple(1000.0, false) << '\n';
}
```

ผลลัพธ์:

```
แบบ over-engineered (VIP): 800
แบบ over-engineered (ปกติ): 1000
แบบเรียบง่าย (VIP): 800
แบบเรียบง่าย (ปกติ): 1000
```

ทั้งสองแบบให้ผลลัพธ์เหมือนกันทุกประการ แต่แบบ over-engineered เพิ่ม **3 class, virtual
function 1 ตัว, และ dynamic dispatch** โดยไม่ได้ประโยชน์อะไรเพิ่มขึ้นเลย เพราะเงื่อนไขมีแค่
2 ทางเลือกที่ไม่มีทีท่าว่าจะเพิ่มขึ้น — ถ้าในอนาคตมีส่วนลดแบบที่ 3, 4, 5 ค่อยพิจารณาเปลี่ยนไป
ใช้ Strategy Pattern ตอนนั้นก็ยังไม่สาย (**YAGNI — You Aren't Gonna Need It**, หลักการที่บอก
ว่าอย่าสร้างความยืดหยุ่นสำหรับ requirement ที่ยังไม่มีอยู่จริง)

### คำถาม 4 ข้อที่ควรถามตัวเองก่อนใช้ Pattern ทุกครั้ง

1. **มีปัญหาจริงอยู่ตรงหน้าหรือยัง?** — pattern มีไว้แก้ปัญหาที่มีอยู่จริง ไม่ใช่ปัญหาที่คิดว่า
   "อาจจะเกิดในอนาคต" ถ้ายังไม่มีสัญญาณของปัญหานั้น (เช่น constructor ยังมีแค่ 2-3 พารามิเตอร์
   ไม่จำเป็นต้องใช้ Builder) ให้เขียนโค้ดตรงไปตรงมาก่อน
2. **Pattern นี้ทำให้โค้ดอ่านง่ายขึ้นจริงหรือแค่ซับซ้อนขึ้น?** — ถ้าเพื่อนร่วมทีมต้องเปิดไฟล์
   5-6 ไฟล์เพื่อเข้าใจ logic ที่ควรจะอยู่ในฟังก์ชันเดียว 10 บรรทัด นั่นคือสัญญาณของการ
   over-engineer
3. **ต้นทุนของความยืดหยุ่นนี้คุ้มค่าหรือไม่?** — virtual function, heap allocation ผ่าน
   `unique_ptr`, indirection ผ่าน interface ล้วนมีต้นทุนด้าน performance และความซับซ้อนเสมอ
   (แม้จะเล็กน้อยในเคสส่วนใหญ่) ต้องแลกกับประโยชน์ที่ได้จริง ไม่ใช่แค่ "เผื่อไว้"
4. **ทีมของคุณรู้จัก pattern นี้หรือไม่?** — Design Pattern มีคุณค่าเพราะเป็น **"ภาษากลาง"**
   ถ้าใช้ pattern ที่ซับซ้อนเกินไปในทีมที่ไม่คุ้นเคย จะกลายเป็นภาระในการอ่านโค้ดแทนที่จะช่วย
   สื่อสารให้ง่ายขึ้น

> **หลักการทองคำปิดท้าย Module H**: **Design Pattern เป็นเครื่องมือแก้ปัญหาเฉพาะเจาะจง
> ไม่ใช่เป้าหมายในตัวมันเอง** วิศวกรซอฟต์แวร์ระดับโลกไม่ได้วัดกันที่ "รู้จัก pattern กี่ตัว"
> แต่วัดกันที่ **"รู้ว่าเมื่อไหร่ควรใช้ และเมื่อไหร่ไม่ควรใช้"** โค้ดที่ดีที่สุดมักเป็นโค้ดที่เรียบง่าย
> ที่สุดเท่าที่จะแก้ปัญหาได้ครบถ้วน ไม่ใช่โค้ดที่ยัด pattern เข้าไปให้มากที่สุด

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Observer ที่ไม่ยอม unregister แล้วเก็บ raw pointer หรือ `shared_ptr` ตรงๆ** — ถ้า
   `Subject` เก็บ observer เป็น raw pointer และไม่มีกลไก unregister ที่ทำงานถูกต้อง เมื่อ
   observer ถูกทำลายไปแล้วแต่ subject ยังพยายามเรียกมันอยู่ จะเกิด **dangling pointer** ทันที
   ในทางกลับกัน ถ้าเก็บเป็น `shared_ptr` ตรงๆ (ไม่ใช่ `weak_ptr`) subject จะถือ ownership ไว้
   ทำให้ observer ไม่มีวันถูกทำลายตราบใดที่ยัง subscribe อยู่ — กลายเป็น **memory leak เชิง
   ตรรกะ** (object ยังมีชีวิตอยู่ทั้งที่ไม่ควรมีใครใช้งานมันแล้ว) ทางแก้คือใช้ `weak_ptr` ตาม
   ตัวอย่างในบทเรียน
2. **Decorator ที่ลืมทำให้ทุกชั้นยัง implement interface เดิมครบทุก method** — ถ้า
   `BeverageDecorator` มี virtual method บางตัวที่ derived decorator ลืม override (และ base
   class ไม่ได้ประกาศเป็น pure virtual) จะได้พฤติกรรม default ที่ผิดโดยไม่มี compiler เตือน
   ให้ ควรตรวจสอบว่าทุก method ของ interface ถูก override อย่างถูกต้องในทุกชั้นของ decorator
3. **Composite ที่คำนวณ `sizeBytes()`/traversal ซ้ำโดยไม่จำเป็น (ไม่มี caching)** — ในตัวอย่าง
   `Directory::sizeBytes()` คำนวณใหม่ทุกครั้งที่ถูกเรียกแบบ recursive ถ้า tree มีขนาดใหญ่และ
   ถูกเรียกบ่อย ควรพิจารณา cache ผลลัพธ์และ invalidate cache เมื่อมีการ `add()`/`remove()`
   child ใหม่ เพื่อไม่ให้ performance แย่ลงตามความลึกและขนาดของ tree
4. **Command History ที่โตไม่มีขีดจำกัด** — ตัวอย่าง `RemoteControl::history_` เก็บทุกคำสั่ง
   ไว้ตลอดไปโดยไม่มีการจำกัดขนาด ถ้าโปรแกรมทำงานนาน (เช่น text editor ที่เปิดทิ้งไว้ทั้งวัน)
   `history_` จะใช้ memory เพิ่มขึ้นเรื่อยๆ โค้ด production ควรจำกัดจำนวนคำสั่งสูงสุดที่เก็บไว้
   (เช่น เก็บแค่ 100 คำสั่งล่าสุด แล้วลบคำสั่งเก่าสุดทิ้งเมื่อเกิน)
5. **Iterator ที่ประกาศ `iterator_category` ผิดจากความสามารถจริง** — ถ้า custom iterator
   ประกาศตัวเองเป็น `random_access_iterator_tag` ทั้งที่ implement `operator+`, `operator-`,
   `operator[]` ไม่ครบ STL algorithm บางตัวที่ตรวจสอบ category จะเรียกใช้ operation ที่ไม่มี
   จริง ทำให้ compile error ที่ error message อ่านยากมาก (เพราะ error เกิดลึกใน STL header)
   ควรประกาศ category ให้ตรงกับความสามารถจริงเสมอ (ในตัวอย่างบทนี้คือ `forward_iterator_tag`
   เพราะ implement แค่ `operator++`)
6. **ยัด Pattern เข้าไปทุกที่โดยไม่มีปัญหาจริงรองรับ** — ดังที่อธิบายในหัวข้อก่อนหน้า นี่คือ
   pitfall ที่ร้ายแรงที่สุดในภาพรวม เพราะทำให้โค้ดเบสทั้งระบบซับซ้อนเกินความจำเป็น ยากต่อการ
   onboard วิศวกรใหม่ และยากต่อการแก้ไขในระยะยาว ทั้งที่จุดประสงค์ดั้งเดิมของ pattern คือทำให้
   โค้ดง่ายขึ้น ไม่ใช่ยากขึ้น

---

## แบบฝึกหัดท้ายบท

1. เขียน Adapter Pattern ที่แปลง class `LegacyLogger` (มี method `writeLog(const char* msg,
   int severity)` โดย `severity` คือ 0=info, 1=warning, 2=error) ให้ตรงกับ interface ใหม่
   `Logger` ที่มี `info(const std::string&)`, `warning(const std::string&)`,
   `error(const std::string&)`
2. ขยายตัวอย่าง Decorator (`Beverage`) ในบทเรียน โดยเพิ่ม `VanillaDecorator` ที่เพิ่มราคา 8
   บาท และคำอธิบาย " + วานิลลา" แล้วทดลองประกอบเครื่องดื่มที่มีทั้งนม วานิลลา และวิปครีม
3. เขียน Composite Pattern สำหรับโครงสร้างเมนูร้านอาหาร โดยมี `MenuItem` (leaf, มีชื่อและ
   ราคา) และ `MenuCategory` (composite, เก็บ `MenuItem` หรือ `MenuCategory` ย่อยได้) แล้ว
   คำนวณราคารวมทั้งเมนู
4. ขยายตัวอย่าง Observer (`StockTicker`) ในบทเรียน โดยเพิ่ม observer ชนิดใหม่
   `PriceAlertObserver` ที่จะพิมพ์ข้อความเตือนเมื่อราคาสูงเกินค่าที่กำหนดไว้ตอนสร้าง
   (เช่น เตือนเมื่อราคาสูงกว่า 40 บาท)
5. เขียน Command Pattern สำหรับเครื่องคิดเลขอย่างง่าย ที่รองรับคำสั่ง `AddCommand` และ
   `SubtractCommand` พร้อม `undo()` โดยมี `Calculator` เป็น Receiver ที่เก็บค่าปัจจุบัน
   (`currentValue`)
6. อธิบายด้วยคำพูดตัวเอง (ไม่ต้องเขียนโค้ด) ว่าทำไม GoF Iterator Pattern ถึงเป็นรากฐานของ
   ทุก container ใน STL และยกตัวอย่างว่าถ้า STL **ไม่มี** iterator pattern โค้ดที่ใช้
   `std::vector`, `std::list`, `std::map` ร่วมกับ `std::sort`, `std::find` จะซับซ้อนขึ้น
   อย่างไร

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>
#include <string>

class LegacyLogger {
public:
    void writeLog(const char* msg, int severity) {
        const char* label = severity == 0 ? "INFO" : severity == 1 ? "WARN" : "ERROR";
        std::cout << "[Legacy][" << label << "] " << msg << '\n';
    }
};

class Logger {
public:
    virtual ~Logger() = default;
    virtual void info(const std::string& msg) = 0;
    virtual void warning(const std::string& msg) = 0;
    virtual void error(const std::string& msg) = 0;
};

class LegacyLoggerAdapter : public Logger {
public:
    void info(const std::string& msg) override { legacy_.writeLog(msg.c_str(), 0); }
    void warning(const std::string& msg) override { legacy_.writeLog(msg.c_str(), 1); }
    void error(const std::string& msg) override { legacy_.writeLog(msg.c_str(), 2); }

private:
    LegacyLogger legacy_;
};

int main() {
    LegacyLoggerAdapter logger;
    logger.info("ระบบเริ่มทำงาน");
    logger.warning("พื้นที่ดิสก์เหลือน้อย");
    logger.error("เชื่อมต่อฐานข้อมูลล้มเหลว");
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 legacy_adapter.cpp -o legacy_adapter
./legacy_adapter
```

ผลลัพธ์:

```
[Legacy][INFO] ระบบเริ่มทำงาน
[Legacy][WARN] พื้นที่ดิสก์เหลือน้อย
[Legacy][ERROR] เชื่อมต่อฐานข้อมูลล้มเหลว
```

### แนวทางเฉลยข้อ 5

```cpp
#include <iostream>
#include <memory>
#include <vector>

// ---------- Receiver ----------
class Calculator {
public:
    void add(double x) {
        current_ += x;
        std::cout << "current = " << current_ << '\n';
    }
    void subtract(double x) {
        current_ -= x;
        std::cout << "current = " << current_ << '\n';
    }
    double current() const { return current_; }

private:
    double current_ = 0.0;
};

// ---------- Command interface ----------
class CalculatorCommand {
public:
    virtual ~CalculatorCommand() = default;
    virtual void execute() = 0;
    virtual void undo() = 0;
};

class AddCommand : public CalculatorCommand {
public:
    AddCommand(Calculator& calc, double amount) : calc_(calc), amount_(amount) {}
    void execute() override { calc_.add(amount_); }
    void undo() override { calc_.subtract(amount_); }

private:
    Calculator& calc_;
    double amount_;
};

class SubtractCommand : public CalculatorCommand {
public:
    SubtractCommand(Calculator& calc, double amount) : calc_(calc), amount_(amount) {}
    void execute() override { calc_.subtract(amount_); }
    void undo() override { calc_.add(amount_); }

private:
    Calculator& calc_;
    double amount_;
};

class CommandInvoker {
public:
    void run(std::unique_ptr<CalculatorCommand> command) {
        command->execute();
        history_.push_back(std::move(command));
    }

    void undoLast() {
        if (history_.empty()) return;
        history_.back()->undo();
        history_.pop_back();
    }

private:
    std::vector<std::unique_ptr<CalculatorCommand>> history_;
};

int main() {
    Calculator calc;
    CommandInvoker invoker;

    invoker.run(std::make_unique<AddCommand>(calc, 10.0));       // current = 10
    invoker.run(std::make_unique<AddCommand>(calc, 5.0));        // current = 15
    invoker.run(std::make_unique<SubtractCommand>(calc, 3.0));   // current = 12

    std::cout << "-- undo ล่าสุด --\n";
    invoker.undoLast();   // undo SubtractCommand(3) -> current = 15

    std::cout << "ค่าสุดท้าย: " << calc.current() << '\n';
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 calculator_command.cpp -o calculator_command
./calculator_command
```

ผลลัพธ์:

```
current = 10
current = 15
current = 12
-- undo ล่าสุด --
current = 15
ค่าสุดท้าย: 15
```

(ข้อ 2, 3, 4 และ 6 ให้ผู้เรียนลองทำเองตามแนวทางของตัวอย่างในบทเรียน — สิ่งสำคัญคือทุกคำตอบ
ต้องคอมไพล์ผ่านด้วย `-Wall -Wextra -Wpedantic -std=c++17` โดยไม่มี warning ใดๆ เลย และ
ควรลองรันจริงเพื่อตรวจสอบผลลัพธ์ก่อนถือว่าทำเสร็จ)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจจุดประสงค์ของ **Structural Pattern** (ประกอบโครงสร้าง) และ **Behavioral Pattern**
  (สื่อสาร/พฤติกรรม) ซึ่งเป็นอีก 2 หมวดหมู่หลักของ GoF ต่อจาก Creational ใน Part 97
- ใช้ **Adapter** เชื่อมต่อ interface เก่ากับใหม่, **Decorator** เพิ่มพฤติกรรมแบบ dynamic,
  **Facade** ซ่อนความซับซ้อนของ subsystem, และ **Composite** จัดการโครงสร้างต้นไม้
- ใช้ **Observer** พร้อม `std::weak_ptr` สร้างระบบแจ้งเตือนที่ปลอดภัยจาก memory leak และ
  dangling pointer — พื้นฐานสำคัญก่อนเข้าสู่ระบบ event-driven ใน Module I
- ใช้ **Strategy** สลับ algorithm ที่ runtime ทั้งแบบ class hierarchy และแบบ `std::function`
- ใช้ **Command** ห่อคำสั่งเป็น object เพื่อรองรับ undo/redo
- implement **Iterator** ของตัวเองที่เข้ากันได้กับ STL algorithm และเข้าใจว่า STL ทั้งหมด
  สร้างอยู่บนหลักการของ GoF Iterator Pattern
- เรียนรู้บทเรียนสำคัญที่สุดของ Module H เรื่อง Design Pattern: **อย่า over-engineer**
  — pattern คือเครื่องมือแก้ปัญหาเฉพาะเจาะจง ไม่ใช่เป้าหมายในตัวเอง โค้ดที่ดีที่สุดคือโค้ดที่
  เรียบง่ายที่สุดเท่าที่จะแก้ปัญหาได้ครบถ้วน

Module H ทั้งหมด (Part 91-98) ได้พาเราผ่าน Build Systems (CMake), Package Management,
Unit Testing, CI/CD, Static Analysis, Sanitizer และ Design Pattern มาครบถ้วน — นี่คือชุด
ทักษะที่แยกวิศวกร C++ มืออาชีพออกจากผู้ที่แค่ "เขียนโค้ดให้รัน" ได้อย่างชัดเจน

ใน **Module I: Web Development** ที่กำลังจะเริ่มต้นใน **Part 99** เราจะนำทักษะทั้งหมดที่
สั่งสมมาตลอด 98 Part — ตั้งแต่ pointer, OOP, smart pointer, concurrency, ไปจนถึง Design
Pattern ที่เพิ่งเรียนจบ — มาประยุกต์สร้างเว็บเซิร์ฟเวอร์และ REST API ด้วย C++ ตั้งแต่ระดับ
Socket ดิบๆ ไปจนถึง Framework ระดับ Production จริง

**ต่อไป:** [Part 99 — ภาพรวม Web Development ด้วย C/C++](./part-099-web-dev-overview.md)
