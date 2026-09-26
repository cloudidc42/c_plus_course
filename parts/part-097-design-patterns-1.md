# Part 97: Design Pattern ใน C++ (ตอนที่ 1): Creational Patterns (Step 769–776)

> Module H — Build Systems, Testing และ Tooling | Part 97 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 769–776
> Part ก่อนหน้า: [Part 96 — Sanitizer และ Code Coverage](./part-096-sanitizers-coverage.md) | Part ถัดไป: [Part 98 — Design Pattern ใน C++ (ตอนที่ 2): Structural และ Behavioral Patterns](./part-098-design-patterns-2.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า Design Pattern คืออะไร แก้ปัญหาอะไร และทำไมวงการซอฟต์แวร์ถึงยังพูดถึงหนังสือ
   Gang of Four (GoF) ปี 1994 มาจนถึงทุกวันนี้
2. จำแนก Design Pattern ทั้ง 3 หมวดหมู่ (Creational, Structural, Behavioral) และบอกได้ว่า
   pattern แต่ละตัวที่จะเรียนอยู่หมวดไหน แก้ปัญหาอะไร
3. ทบทวนและต่อยอด **Singleton Pattern** จาก Part 53 ให้ลึกขึ้น เข้าใจปัญหา thread-safety
   ของ Singleton แบบเก่า (double-checked locking) และรู้จัก `std::call_once` เป็นทางเลือก
4. ออกแบบและ implement **Factory Method** และ **Abstract Factory** เพื่อแยก "โค้ดที่สร้าง
   object" ออกจาก "โค้ดที่ใช้ object" ด้วย `unique_ptr` แทน raw pointer
5. ใช้ **Builder Pattern** แก้ปัญหา constructor ที่มีพารามิเตอร์เยอะเกินไป (telescoping
   constructor) ด้วย fluent interface และเข้าใจบทบาทของ Director
6. ใช้ **Prototype Pattern** สร้าง object ใหม่จากการ clone object ต้นแบบ แทนการสร้างใหม่
   ทั้งหมด และเชื่อมโยงกับ copy constructor ที่เรียนมาตั้งแต่ Module C
7. อ่านและวาด UML class diagram แบบง่ายเพื่อสื่อสารโครงสร้างของ pattern กับเพื่อนร่วมทีมได้
8. เลือกใช้ Creational Pattern ที่เหมาะสมกับสถานการณ์จริง แทนการใช้ pattern แบบสุ่มหรือใช้
   เพราะ "เท่ห์" โดยไม่มีปัญหาจริงให้แก้

---

## 97.1 Design Pattern คืออะไร และทำไมยังสำคัญในปี 2026 (Step 769)

ตลอด 96 Part ที่ผ่านมา เราเรียนรู้ "อิฐก้อนเล็กๆ" ของภาษา C/C++ ไปทีละก้อน ตั้งแต่ตัวแปร
ไปจนถึง class, inheritance, template, smart pointer, concurrency ฯลฯ แต่เมื่อโปรแกรมเมอร์
หลายพันคนทั่วโลกเอาอิฐเหล่านี้ไปสร้างระบบซอฟต์แวร์จริง พวกเขาพบว่า **ปัญหาการออกแบบบางแบบ
เกิดขึ้นซ้ำๆ กัน** ในบริบทที่ต่างกันโดยสิ้นเชิง เช่น:

- "ฉันอยากให้มี object แค่ตัวเดียวในทั้งโปรแกรม" (เจอทั้งใน Logger, Configuration, Database
  Connection Pool, Game Manager)
- "ฉันอยากให้ constructor รับพารามิเตอร์ optional เยอะๆ โดยไม่ทำให้โค้ดอ่านยาก" (เจอทั้งใน
  HTTP Request, SQL Query Builder, GUI Widget)
- "ฉันอยากให้ object A รู้ว่า object B เปลี่ยนแปลง โดยไม่ผูก A กับ B แน่นเกินไป" (เจอทั้งใน
  GUI Event, Stock Price Update, Game Event System)

ในปี 1994 ทีมนักออกแบบซอฟต์แวร์ 4 คน — **Erich Gamma, Richard Helm, Ralph Johnson,
John Vlissides** — ตีพิมพ์หนังสือ *Design Patterns: Elements of Reusable Object-Oriented
Software* รวบรวม **23 pattern** ที่เป็น "solution ที่พิสูจน์แล้วซ้ำแล้วซ้ำเล่า" สำหรับปัญหาการ
ออกแบบที่เกิดขึ้นซ้ำๆ เหล่านี้ ผู้เขียนทั้งสี่คนถูกเรียกกันติดปากว่า **Gang of Four (GoF)** และ
หนังสือเล่มนี้ถูกเรียกสั้นๆ ว่า "หนังสือ GoF" มาจนถึงทุกวันนี้

### นิยามที่ชัดเจนของ Design Pattern

> **Design Pattern** คือ **"คำตอบที่มีชื่อเรียก" (named solution)** สำหรับปัญหาการออกแบบที่
> เกิดขึ้นซ้ำๆ ในบริบทหนึ่งๆ ไม่ใช่โค้ดสำเร็จรูปที่ copy-paste ได้ แต่เป็น **แนวคิดโครงสร้าง**
> ที่ต้องปรับให้เข้ากับภาษาและปัญหาของตัวเองเสมอ

ประเด็นสำคัญ 3 ข้อที่มือใหม่มักเข้าใจผิด:

1. **Pattern ไม่ใช่โค้ด** — มันคือ "รูปแบบความสัมพันธ์ระหว่าง class/object" หนังสือ GoF อธิบาย
   pattern ด้วย UML diagram และคำอธิบาย ไม่ใช่ library ที่ import มาใช้ได้ตรงๆ
2. **Pattern ไม่ใช่เป้าหมาย** — pattern มีไว้ **แก้ปัญหาที่มีอยู่จริง** ไม่ใช่สิ่งที่ต้องยัดเข้าไปใน
   ทุกโปรเจกต์เพื่อให้ดูเป็นมืออาชีพ (เราจะพูดเรื่องนี้ซ้ำอีกครั้งท้าย Part 98)
3. **ทำไมต้องมีชื่อ** — เมื่อทีมพูดว่า "ใช้ Factory ตรงนี้" ทุกคนที่รู้จัก pattern จะเข้าใจโครงสร้าง
   ทันทีโดยไม่ต้องอธิบายยาว นี่คือคุณค่าที่แท้จริงของการมี "ภาษากลาง" (shared vocabulary)
   ระหว่างวิศวกรซอฟต์แวร์ทั่วโลก

### 3 หมวดหมู่หลักของ GoF Pattern

หนังสือ GoF แบ่ง 23 pattern ออกเป็น 3 หมวดตาม "จุดประสงค์" (purpose):

```
┌─────────────────────────────────────────────────────────────────┐
│                     Design Pattern (GoF, 23 แบบ)                 │
├───────────────────┬───────────────────────┬─────────────────────┤
│    Creational      │      Structural        │      Behavioral     │
│  (การสร้าง object)  │   (การประกอบโครงสร้าง)  │  (การสื่อสาร/พฤติกรรม) │
├───────────────────┼───────────────────────┼─────────────────────┤
│ Singleton          │ Adapter               │ Observer            │
│ Factory Method     │ Decorator             │ Strategy            │
│ Abstract Factory   │ Facade                │ Command             │
│ Builder            │ Composite             │ Iterator            │
│ Prototype          │ (Proxy, Bridge, ...)  │ (State, Visitor,...)│
├───────────────────┴───────────────────────┴─────────────────────┤
│         ← Part 97 (Part นี้)  │  Part 98 (Structural + Behavioral)│
└───────────────────────────────────────────────────────────────────┘
```

- **Creational (สร้าง object)**: เกี่ยวกับ "การสร้าง object อย่างไรให้ยืดหยุ่นและปลอดภัย"
  โดยไม่ผูกโค้ดผู้ใช้เข้ากับ concrete class ที่ถูกสร้างมากเกินไป — หัวข้อของ Part นี้
- **Structural (ประกอบโครงสร้าง)**: เกี่ยวกับ "การรวม class/object เข้าด้วยกันเป็นโครงสร้าง
  ที่ใหญ่ขึ้น" โดยยังคงยืดหยุ่นและมีประสิทธิภาพ — เรียนใน Part 98
- **Behavioral (พฤติกรรม/การสื่อสาร)**: เกี่ยวกับ "การกระจายความรับผิดชอบและการสื่อสาร
  ระหว่าง object" — เรียนใน Part 98 เช่นกัน

ใน Part นี้เราจะเจาะลึก 5 Creational Pattern ที่สำคัญที่สุดและใช้บ่อยที่สุดในโค้ด C++ จริง:
**Singleton, Factory Method, Abstract Factory, Builder, Prototype** ทุกตัวอย่างจะเขียนด้วย
แนวคิด Modern C++ โดยใช้ `std::unique_ptr`/`std::shared_ptr` จาก Part 67 แทน raw pointer
และ `new`/`delete` ตรงๆ แบบที่หนังสือ GoF ต้นฉบับ (ซึ่งเขียนก่อน C++11 จะมี smart pointer)
เคยใช้

> **หมายเหตุเรื่อง UML**: diagram ในบทเรียนนี้เป็น ASCII art แบบง่ายที่สื่อความสัมพันธ์หลัก
> (inheritance ด้วยลูกศรกลวง `--▷`, composition/aggregation ด้วยเส้นและเพชร) ไม่ใช่ UML
> มาตรฐานเป๊ะทุกกระเบียดนิ้ว แต่เพียงพอสำหรับสื่อสารโครงสร้างในทีมงานจริง

---

## 97.2 Singleton Pattern เจาะลึก: Thread-Safety และข้อควรระวัง (Step 770)

ใน **Part 53.5** เราเคยแนะนำ Singleton Pattern แบบสั้นๆ ผ่าน Meyer's Singleton ไปแล้ว
หัวข้อนี้จะทบทวนแนวคิดหลักอย่างรวบรัด แล้วเจาะลึกในสิ่งที่ Part 53 ยังไม่ได้พูดถึง: ปัญหา
thread-safety ในอดีต และทางเลือกอื่นเมื่อ Meyer's Singleton ไม่เพียงพอ

### ทบทวน: Singleton คืออะไร แก้ปัญหาอะไร

**Singleton** รับประกัน 2 อย่าง:

1. Class นั้นมี **instance ได้เพียงตัวเดียว** ตลอดอายุโปรแกรม
2. มี **จุดเข้าถึงสากล (global access point)** ให้เรียกใช้ instance นั้นจากที่ไหนก็ได้

```
┌───────────────────────┐
│      AppConfig         │
├───────────────────────┤
│ - instance: AppConfig* │ (static, private)
├───────────────────────┤
│ - AppConfig()          │ (private constructor)
│ + instance(): AppConfig&│ (public static method — จุดเข้าถึงเดียว)
└───────────────────────┘
        ▲
        │ ทุกที่ในโปรแกรมเรียกผ่านจุดเดียวกัน
   ┌────┴────┬─────────┬─────────┐
 main()   ModuleA   ModuleB   ModuleC
```

### Meyer's Singleton: มาตรฐานของ Modern C++

```cpp
#include <iostream>
#include <map>
#include <string>

class AppConfig {
public:
    AppConfig(const AppConfig&) = delete;
    AppConfig& operator=(const AppConfig&) = delete;

    static AppConfig& instance() {
        static AppConfig cfg;   // สร้างครั้งแรกตอนถูกเรียกครั้งแรกเท่านั้น (lazy init)
        return cfg;             // C++11 การันตีว่าการสร้างนี้ thread-safe
    }

    void set(const std::string& key, const std::string& value) {
        settings_[key] = value;
    }

    std::string get(const std::string& key) const {
        auto it = settings_.find(key);
        return it != settings_.end() ? it->second : std::string{"(ไม่พบ)"};
    }

private:
    AppConfig() { settings_["env"] = "production"; }
    ~AppConfig() = default;

    std::map<std::string, std::string> settings_;
};

void printDatabaseHost() {
    // เรียกจากที่ไหนในโปรแกรมก็ได้ โดยไม่ต้องส่ง AppConfig& ผ่านทุกฟังก์ชัน
    std::cout << "env = " << AppConfig::instance().get("env") << '\n';
}

int main() {
    AppConfig::instance().set("db_host", "10.0.0.5");

    printDatabaseHost();
    std::cout << "db_host = " << AppConfig::instance().get("db_host") << '\n';

    std::cout << std::boolalpha
              << "instance() คืน object เดียวกันเสมอ: "
              << (&AppConfig::instance() == &AppConfig::instance()) << '\n';
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 app_config.cpp -o app_config
./app_config
```

ผลลัพธ์:

```
env = production
db_host = 10.0.0.5
instance() คืน object เดียวกันเสมอ: true
```

**เหตุผลที่ `static AppConfig cfg;` ภายในฟังก์ชัน `instance()` เป็น thread-safe ตั้งแต่ C++11**:
มาตรฐาน C++11 กำหนดไว้ใน **[stmt.dcl]** ว่าถ้าหลาย thread เรียกฟังก์ชันที่มี static local
variable พร้อมกันเป็นครั้งแรก compiler ต้อง generate โค้ดที่ป้องกัน race condition ให้เองโดย
อัตโนมัติ (ผ่านกลไกคล้าย mutex ที่ซ่อนอยู่ภายใน) โปรแกรมเมอร์ **ไม่ต้องเขียน mutex เอง**
เพื่อป้องกันการสร้าง instance ซ้ำจากหลาย thread — นี่คือเหตุผลที่ Meyer's Singleton (ตั้งชื่อ
ตาม **Scott Meyers** ผู้เผยแพร่เทคนิคนี้อย่างกว้างขวางใน *Effective C++*) กลายเป็นวิธีมาตรฐาน
ของ Modern C++

### ปัญหาในอดีต: Double-Checked Locking (Anti-Pattern)

ก่อน C++11 จะมี memory model ที่ชัดเจน โปรแกรมเมอร์จำนวนมากพยายามเขียน Singleton แบบ
thread-safe ด้วยมือโดยใช้เทคนิคที่เรียกว่า **Double-Checked Locking** ดังนี้:

```cpp
#include <iostream>
#include <mutex>

// *** ตัวอย่างเชิงประวัติศาสตร์เท่านั้น: Double-Checked Locking แบบเดิม ***
// โค้ดนี้ "คอมไพล์ผ่านและรันได้" แต่มีปัญหา thread-safety ที่ compiler ไม่เตือนให้
// (สมัยก่อน C++11 ยังไม่มี memory model ที่ชัดเจน การ reorder คำสั่งของ CPU/compiler
//  อาจทำให้ thread อื่นเห็น pointer ที่ไม่ใช่ null แต่ object ข้างในยัง construct ไม่เสร็จ)
class LegacyResource {
public:
    static LegacyResource* getInstance() {
        if (instance_ == nullptr) {                 // check ครั้งที่ 1 (ไม่ล็อก เพื่อความเร็ว)
            std::lock_guard<std::mutex> lock(mutex_);
            if (instance_ == nullptr) {              // check ครั้งที่ 2 (ล็อกแล้ว)
                instance_ = new LegacyResource();    // <-- จุดที่เป็นปัญหาในยุคก่อน C++11
            }
        }
        return instance_;
    }

    int value() const { return value_; }

private:
    LegacyResource() : value_(42) {}
    int value_;

    static LegacyResource* instance_;
    static std::mutex mutex_;
};

LegacyResource* LegacyResource::instance_ = nullptr;
std::mutex LegacyResource::mutex_;

int main() {
    std::cout << "value = " << LegacyResource::getInstance()->value() << '\n';
}
```

ผลลัพธ์ (รันแบบ single-thread จะไม่มีปัญหาให้เห็น):

```
value = 42
```

**ทำไมโค้ดนี้ถึงเป็นปัญหา** (แม้จะคอมไพล์ผ่านและรันได้ปกติในตัวอย่างข้างต้น): บรรทัด
`instance_ = new LegacyResource();` จริงๆ แล้วประกอบด้วย 3 ขั้นตอนซ่อนอยู่:

```
(1) จองหน่วยความจำสำหรับ LegacyResource
(2) เรียก constructor สร้าง object ในหน่วยความจำนั้น
(3) ให้ instance_ ชี้ไปที่หน่วยความจำนั้น
```

ก่อน C++11 compiler/CPU **ได้รับอนุญาตให้ reorder ขั้นตอน (2) และ (3) สลับกันได้** เพื่อ
optimize ความเร็ว ถ้า thread A ทำถึงขั้นตอน (3) ก่อน (2) เสร็จ แล้ว thread B เข้ามาเช็ค
`instance_ == nullptr` ที่ check ครั้งที่ 1 พอดี thread B จะเห็นว่า `instance_` ไม่ใช่ nullptr
แล้ว (เพราะขั้นตอน 3 ทำไปแล้ว) จึงคืน pointer นั้นกลับไปใช้งานทันที **ทั้งที่ constructor ยัง
ทำงานไม่เสร็จ** ทำให้เกิด Undefined Behavior ที่ตรวจจับได้ยากมากเพราะเกิดเป็นครั้งคราวเท่านั้น
ขึ้นอยู่กับจังหวะของ CPU

> **บทเรียนสำคัญ**: อย่าพยายามเขียน thread-safe Singleton ด้วยมือด้วยเทคนิค double-checked
> locking แบบดิบๆ อีกต่อไปในโค้ดใหม่ ใช้ **Meyer's Singleton** (static local variable) เป็น
> ค่าเริ่มต้นเสมอ เพราะมาตรฐาน C++11 ขึ้นไปรับประกัน thread-safety ให้ฟรีโดยไม่ต้องเขียน
> mutex เอง

### ทางเลือกอื่น: std::call_once สำหรับทรัพยากรที่ต้องควบคุมช่วงชีวิตเอง

Meyer's Singleton เหมาะกับเกือบทุกกรณี แต่บางครั้งเราต้องการควบคุมว่า **เมื่อไหร่ instance
จะถูกทำลาย** อย่างชัดเจน (เช่น connection pool ที่ต้อง cleanup ก่อน object อื่นที่ใช้มัน) หรือ
ต้องการ initialize ผ่าน pointer เพื่อเลี่ยงปัญหา static destruction order ในบางสถาปัตยกรรม
กรณีนี้ใช้ `std::call_once` ร่วมกับ `std::unique_ptr`:

```cpp
#include <iostream>
#include <memory>
#include <mutex>
#include <string>

// Singleton สำหรับทรัพยากรที่ "หนัก" และต้องการควบคุมช่วงชีวิตอย่างชัดเจน (เช่น connection pool)
// ใช้ std::call_once + std::unique_ptr แทนการเขียน double-checked locking เอง
class ConnectionPool {
public:
    ConnectionPool(const ConnectionPool&) = delete;
    ConnectionPool& operator=(const ConnectionPool&) = delete;

    static ConnectionPool& instance() {
        std::call_once(initFlag_, []() {
            instance_.reset(new ConnectionPool());   // การันตีสร้างครั้งเดียว thread-safe จริง
        });
        return *instance_;
    }

    std::string describe() const {
        return "ConnectionPool(maxConnections=" + std::to_string(maxConnections_) + ")";
    }

private:
    ConnectionPool() : maxConnections_(10) {
        std::cout << "[ConnectionPool] กำลังเชื่อมต่อฐานข้อมูล...\n";
    }

    int maxConnections_;

    static std::unique_ptr<ConnectionPool> instance_;
    static std::once_flag initFlag_;
};

std::unique_ptr<ConnectionPool> ConnectionPool::instance_ = nullptr;
std::once_flag ConnectionPool::initFlag_;

int main() {
    std::cout << ConnectionPool::instance().describe() << '\n';
    std::cout << ConnectionPool::instance().describe() << '\n'; // ไม่สร้างซ้ำ ไม่พิมพ์ log อีก
}
```

ผลลัพธ์:

```
[ConnectionPool] กำลังเชื่อมต่อฐานข้อมูล...
ConnectionPool(maxConnections=10)
ConnectionPool(maxConnections=10)
```

`std::call_once` (ใน header `<mutex>`) รับประกันว่า lambda ข้างในจะถูกเรียก **เพียงครั้งเดียว
เท่านั้นตลอดโปรแกรม** ไม่ว่าจะมีกี่ thread เรียก `instance()` พร้อมกันก็ตาม โดยไม่มีปัญหา
reorder แบบ double-checked locking เพราะ `std::call_once`/`std::once_flag` ถูกออกแบบมา
พร้อม memory model ของ C++11 ตั้งแต่ต้น — ใช้ `unique_ptr` แทน raw pointer ทำให้ไม่ต้องเขียน
`delete` เองที่ไหนเลย และยังสอดคล้องกับหลัก RAII จาก Part 68

### เมื่อไหร่ควรใช้แบบไหน

| สถานการณ์ | แนะนำ |
|---|---|
| ส่วนใหญ่ (Logger, Config, Registry) | **Meyer's Singleton** (`static` local variable) — สั้น ปลอดภัย ไม่ต้องคิดมาก |
| ต้องการ custom deleter หรือควบคุม destruction order เอง | `std::call_once` + `unique_ptr` |
| ต้อง reset instance ระหว่าง unit test | ทั้งสองแบบมีปัญหานี้เหมือนกัน (ดู Common Pitfalls) |

---

## 97.3 Factory Method Pattern (Step 771)

### ปัญหาที่ Factory Method แก้

สมมติเรามีแอปพลิเคชันที่ต้องสร้างเอกสารหลายชนิด (Text, Spreadsheet, Presentation) และ
โค้ดส่วนที่เหลือของแอป (การเปิด บันทึก แสดงผล) **ทำงานเหมือนกันทุกชนิดเอกสาร** ยกเว้นแค่
"ขั้นตอนการสร้างเอกสารใหม่" ที่ต่างกัน ถ้าเราเขียน `if (type == "text") ... else if (type ==
"spreadsheet") ...` กระจายอยู่ทั่วโค้ด ทุกครั้งที่เพิ่มเอกสารชนิดใหม่จะต้องไล่แก้ทุกจุดที่มี
`if-else` แบบนี้ — ขัดกับ **Open/Closed Principle** (เปิดให้ขยาย ปิดไม่ให้แก้ไข) ที่เรียนใน
Module D

**Factory Method** แก้ปัญหานี้โดยประกาศ **"factory method"** เป็น virtual function ใน base
class แล้วปล่อยให้แต่ละ subclass ตัดสินใจว่าจะสร้าง concrete product ชนิดไหน โดยที่ logic
ส่วนที่เหมือนกัน (template method) เขียนไว้ที่เดียวใน base class

```
┌────────────────────┐              ┌──────────────────┐
│     Application     │ (Creator)    │     Document      │ (Product)
├────────────────────┤              ├──────────────────┤
│ +newDocumentWorkflow()│  สร้าง ──▷  │ +open()           │
│ #createDocument()*  │ (abstract)   └──────────────────┘
└────────────────────┘                       △
        △                                    │
   ┌────┴────────┐                  ┌────────┴─────────┐
┌──────────────┐ ┌──────────────────┐ ┌──────────────┐ ┌─────────────────────┐
│TextApplication│ │SpreadsheetApp... │ │TextDocument   │ │SpreadsheetDocument  │
├──────────────┤ ├──────────────────┤ └──────────────┘ └─────────────────────┘
│#createDocument()│ #createDocument()│
│ -> TextDocument │ -> SpreadsheetDoc│
└──────────────┘ └──────────────────┘
```

```cpp
#include <iostream>
#include <memory>
#include <string>

// ---------- Product (สินค้าที่ถูกสร้าง) ----------
class Document {
public:
    virtual ~Document() = default;
    virtual std::string type() const = 0;
    virtual void open() const {
        std::cout << "เปิดไฟล์ " << type() << " ด้วยโปรแกรมที่เหมาะสม\n";
    }
};

class TextDocument : public Document {
public:
    std::string type() const override { return "Text Document (.txt)"; }
};

class SpreadsheetDocument : public Document {
public:
    std::string type() const override { return "Spreadsheet (.csv)"; }
};

// ---------- Creator (ตัวประกาศ Factory Method) ----------
// จุดสำคัญของ Factory Method: ตัว Application (base class) ไม่รู้จักชนิดคอนกรีตของ
// Document เลย มันรู้แค่ "ต้องมีคนสร้าง Document ให้ฉัน" แล้วปล่อยให้ subclass ตัดสินใจ
class Application {
public:
    virtual ~Application() = default;

    // Template Method ที่เรียก factory method ภายใน — ทำงานร่วมกับ Document
    // โดยไม่ผูกติดกับชนิดคอนกรีตของมันเลย
    void newDocumentWorkflow() {
        std::unique_ptr<Document> doc = createDocument();   // <-- Factory Method
        std::cout << "สร้างเอกสารใหม่ในแอป: ";
        doc->open();
    }

protected:
    virtual std::unique_ptr<Document> createDocument() const = 0;   // Factory Method
};

class TextApplication : public Application {
protected:
    std::unique_ptr<Document> createDocument() const override {
        return std::make_unique<TextDocument>();
    }
};

class SpreadsheetApplication : public Application {
protected:
    std::unique_ptr<Document> createDocument() const override {
        return std::make_unique<SpreadsheetDocument>();
    }
};

void runApplication(Application& app) {
    app.newDocumentWorkflow();
}

int main() {
    TextApplication textApp;
    SpreadsheetApplication sheetApp;

    runApplication(textApp);
    runApplication(sheetApp);
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 factory_method.cpp -o factory_method
./factory_method
```

ผลลัพธ์:

```
สร้างเอกสารใหม่ในแอป: เปิดไฟล์ Text Document (.txt) ด้วยโปรแกรมที่เหมาะสม
สร้างเอกสารใหม่ในแอป: เปิดไฟล์ Spreadsheet (.csv) ด้วยโปรแกรมที่เหมาะสม
```

จุดสำคัญที่ต้องสังเกต: `createDocument()` เป็น `protected` และ `pure virtual` — บังคับให้
subclass ต้อง override แต่ **ผู้ใช้ภายนอกเรียกไม่ได้โดยตรง** เพราะไม่ใช่ interface ที่ผู้ใช้ควร
ยุ่งเกี่ยวเอง ผู้ใช้เรียกแค่ `newDocumentWorkflow()` ที่เป็น public เท่านั้น การคืนค่าเป็น
`std::unique_ptr<Document>` แทน raw pointer ทำให้ `Application::newDocumentWorkflow()`
ไม่ต้องกังวลเรื่อง memory leak เลย — เมื่อ `doc` หลุด scope memory จะถูกคืนอัตโนมัติ

> **ข้อสังเกตเรื่องคำศัพท์**: หลายคนสับสนระหว่าง "Factory Method" กับ "Simple Factory"
> (บาง textbook เรียก "Static Factory") ซึ่งเป็นแค่ฟังก์ชันเดียวที่มี `if/switch` เลือกสร้าง
> object ตาม parameter — **ไม่ใช่ pattern ใน GoF อย่างเป็นทางการ** แต่เป็น idiom ที่พบบ่อยและ
> มีประโยชน์ในตัวของมันเอง ต่างจาก Factory Method ที่แท้จริงตรงที่ Factory Method ใช้
> **polymorphism ผ่านการ subclass** ในการตัดสินใจ ไม่ใช่ `if/switch`

---

## 97.4 Abstract Factory Pattern (Step 772)

### ปัญหาที่ Abstract Factory แก้: การสร้าง object เป็น "ตระกูล" ที่ต้องเข้าคู่กันเสมอ

ลองจินตนาการแอปที่รองรับหลายธีม (Light/Dark) และแต่ละธีมมี widget หลายชนิด (Button,
Checkbox, ...) ที่ต้อง **มาจากธีมเดียวกันเสมอ** — ถ้าปล่อยให้ client เลือกสร้าง `LightButton`
คู่กับ `DarkCheckbox` แบบผสมกันมั่ว จะได้ UI ที่หน้าตาไม่สอดคล้องกัน (inconsistent)

**Abstract Factory** แก้ปัญหานี้โดยให้ factory หนึ่งตัวรับผิดชอบสร้าง **สินค้าทั้งตระกูล**
(family of related objects) พร้อมกัน แทนที่จะให้ client เลือกประกอบเอง

```
┌────────────────┐         ┌────────┐         ┌──────────┐
│   GUIFactory    │  สร้าง ──▷│ Button │         │ Checkbox │
│ (Abstract Factory)│──┬────└────────┘         └──────────┘
├────────────────┤  │              △                 △
│+createButton()* │  │      ┌──────┴──────┐   ┌───────┴──────┐
│+createCheckbox()*│  │  ┌───────────┐ ┌────────────┐  ...
└────────────────┘  │  │LightButton│ │DarkButton  │
        △            │  └───────────┘ └────────────┘
   ┌────┴────┐        └──── (สร้างคู่กันเสมอ ไม่มีทางผสมข้ามธีม)
┌───────────┐ ┌──────────┐
│LightFactory│ │DarkFactory│
└───────────┘ └──────────┘
```

```cpp
#include <iostream>
#include <memory>
#include <string>

// ---------- Abstract Products (สินค้าแต่ละ "ตระกูล") ----------
class Button {
public:
    virtual ~Button() = default;
    virtual void render() const = 0;
};

class Checkbox {
public:
    virtual ~Checkbox() = default;
    virtual void render() const = 0;
};

// ---------- Concrete Products: ธีม Light ----------
class LightButton : public Button {
public:
    void render() const override { std::cout << "[ปุ่มธีมสว่าง]\n"; }
};

class LightCheckbox : public Checkbox {
public:
    void render() const override { std::cout << "[checkbox ธีมสว่าง]\n"; }
};

// ---------- Concrete Products: ธีม Dark ----------
class DarkButton : public Button {
public:
    void render() const override { std::cout << "[ปุ่มธีมมืด]\n"; }
};

class DarkCheckbox : public Checkbox {
public:
    void render() const override { std::cout << "[checkbox ธีมมืด]\n"; }
};

// ---------- Abstract Factory: สร้างสินค้าทั้ง "ตระกูล" ให้เข้าคู่กันเสมอ ----------
class GUIFactory {
public:
    virtual ~GUIFactory() = default;
    virtual std::unique_ptr<Button> createButton() const = 0;
    virtual std::unique_ptr<Checkbox> createCheckbox() const = 0;
};

class LightFactory : public GUIFactory {
public:
    std::unique_ptr<Button> createButton() const override {
        return std::make_unique<LightButton>();
    }
    std::unique_ptr<Checkbox> createCheckbox() const override {
        return std::make_unique<LightCheckbox>();
    }
};

class DarkFactory : public GUIFactory {
public:
    std::unique_ptr<Button> createButton() const override {
        return std::make_unique<DarkButton>();
    }
    std::unique_ptr<Checkbox> createCheckbox() const override {
        return std::make_unique<DarkCheckbox>();
    }
};

// ---------- Client: ใช้งานผ่าน interface ของ Abstract Factory เท่านั้น ----------
void renderForm(const GUIFactory& factory) {
    std::unique_ptr<Button> button = factory.createButton();
    std::unique_ptr<Checkbox> checkbox = factory.createCheckbox();
    button->render();
    checkbox->render();
}

int main() {
    std::cout << "-- ผู้ใช้เลือกธีมสว่าง --\n";
    LightFactory lightFactory;
    renderForm(lightFactory);

    std::cout << "-- ผู้ใช้เลือกธีมมืด --\n";
    DarkFactory darkFactory;
    renderForm(darkFactory);

    // ข้อดี: ไม่มีทางได้ LightButton คู่กับ DarkCheckbox แบบผสมมั่วเลย
    // เพราะ factory หนึ่งตัวรับผิดชอบสร้างทั้งตระกูลให้เข้าคู่กันเสมอ
}
```

ผลลัพธ์:

```
-- ผู้ใช้เลือกธีมสว่าง --
[ปุ่มธีมสว่าง]
[checkbox ธีมสว่าง]
-- ผู้ใช้เลือกธีมมืด --
[ปุ่มธีมมืด]
[checkbox ธีมมืด]
```

### Factory Method vs. Abstract Factory: ความแตกต่างที่มือใหม่สับสนบ่อยที่สุด

| แง่มุม | Factory Method | Abstract Factory |
|---|---|---|
| สร้าง product กี่ชนิด | ชนิดเดียว (`Document`) | หลายชนิดที่เป็นตระกูลเดียวกัน (`Button` + `Checkbox`) |
| กลไกหลัก | Inheritance + override virtual method 1 ตัว | Composition: object factory ที่มี method สร้างหลายตัว |
| เปลี่ยนพฤติกรรมได้ตอนไหน | ตอน compile (เลือก subclass ของ Creator) | ตอน runtime ก็ได้ (ส่ง factory object ที่ต่างกันเข้าไป) |
| ใช้เมื่อไหร่ | ต้องการให้ subclass ตัดสินใจ "สร้างอะไร" | ต้องการการันตีว่า object หลายตัวมาจาก "ตระกูล" เดียวกัน |

ในทางปฏิบัติ **Abstract Factory มักถูก implement โดยใช้ Factory Method ภายใน** (สังเกตว่า
`createButton()`/`createCheckbox()` แต่ละตัวก็คือ Factory Method ย่อยๆ) — pattern หลายตัว
ทำงานร่วมกันแบบนี้เป็นเรื่องปกติมากในโค้ดจริง ไม่ใช่ทุก pattern จะถูกใช้แบบโดดๆ

---

## 97.5 Builder Pattern: แก้ปัญหา Constructor ที่มีพารามิเตอร์เยอะเกินไป (Step 773)

### ปัญหา: Telescoping Constructor

Builder Pattern เป็น pattern ที่ **สำคัญมากเป็นพิเศษใน C++** เพราะ C++ ไม่มี named
parameter หรือ default keyword argument แบบ Python ทำให้ constructor ที่มีพารามิเตอร์
optional หลายตัวอ่านยากมาก โดยเฉพาะเมื่อพารามิเตอร์เป็นชนิดเดียวกันติดกัน:

```cpp
#include <iostream>
#include <string>

// ปัญหา "Telescoping Constructor": constructor overload หลายตัวจนอ่านไม่รู้เรื่องว่า
// argument แต่ละตัวคืออะไร โดยเฉพาะเมื่อพารามิเตอร์เป็นชนิดเดียวกันติดกันหลายตัว
class HttpRequestOldStyle {
public:
    HttpRequestOldStyle(std::string url, std::string method, std::string body,
                         int timeoutMs, bool followRedirects, bool verifySsl)
        : url_(std::move(url)), method_(std::move(method)), body_(std::move(body)),
          timeoutMs_(timeoutMs), followRedirects_(followRedirects), verifySsl_(verifySsl) {}

    void describe() const {
        std::cout << method_ << " " << url_
                  << " (timeout=" << timeoutMs_ << "ms"
                  << ", redirects=" << std::boolalpha << followRedirects_
                  << ", verifySsl=" << verifySsl_ << ")\n";
    }

private:
    std::string url_;
    std::string method_;
    std::string body_;
    int timeoutMs_;
    bool followRedirects_;
    bool verifySsl_;
};

int main() {
    // อ่านตรงนี้แล้วรู้ไหมว่า true ตัวแรกคือ followRedirects หรือ verifySsl?
    // ต้องเปิดไปดู signature ของ constructor ทุกครั้ง -- นี่คือปัญหาที่ Builder แก้
    HttpRequestOldStyle req("https://api.example.com/users", "POST", "{}", 3000, true, false);
    req.describe();
}
```

ผลลัพธ์:

```
POST https://api.example.com/users (timeout=3000ms, redirects=true, verifySsl=false)
```

โค้ดนี้ **คอมไพล์ผ่านและทำงานถูกต้อง** แต่ปัญหาคือ ณ จุดเรียก `HttpRequestOldStyle(...)`
ไม่มีทางรู้เลยว่า `true, false` ตัวสุดท้ายหมายถึงอะไรโดยไม่เปิดไปดู signature — ยิ่งพารามิเตอร์
ชนิดเดียวกัน (`bool`, `bool`) เรียงติดกัน ยิ่งเสี่ยงต่อการสลับลำดับผิดโดย compiler ไม่มีทาง
เตือนได้เลย (ชนิดข้อมูลตรงกันหมด)

### ทางออก: Builder Pattern ด้วย Fluent Interface

```
┌──────────────────┐  build()   ┌─────────────┐
│HttpRequestBuilder │ ─────────▷ │ HttpRequest  │
├──────────────────┤            └─────────────┘
│+method(m)  -> this&           (friend: เข้าถึง
│+body(b)    -> this&            private member ได้)
│+timeout(t) -> this&
│+noRedirects() -> this&
│+build() -> unique_ptr<HttpRequest>
└──────────────────┘
  (เมธอดแต่ละตัว "return *this" เพื่อให้ chain ต่อกันได้ = fluent interface)
```

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <utility>

// ---------- Product ----------
class HttpRequest {
public:
    void describe() const {
        std::cout << method_ << " " << url_
                  << " (timeout=" << timeoutMs_ << "ms"
                  << ", redirects=" << std::boolalpha << followRedirects_
                  << ", verifySsl=" << verifySsl_ << ")\n";
        if (!body_.empty()) {
            std::cout << "  body: " << body_ << '\n';
        }
    }

    // ให้สิทธิ์ HttpRequestBuilder เข้าถึง private member ได้โดยตรงตอนประกอบร่าง
    friend class HttpRequestBuilder;

private:
    std::string url_;
    std::string method_ = "GET";
    std::string body_;
    int timeoutMs_ = 5000;
    bool followRedirects_ = true;
    bool verifySsl_ = true;
};

// ---------- Builder: ประกอบร่าง HttpRequest ทีละส่วนด้วยชื่อ method ที่สื่อความหมาย ----------
class HttpRequestBuilder {
public:
    explicit HttpRequestBuilder(std::string url) {
        request_ = std::make_unique<HttpRequest>();
        request_->url_ = std::move(url);
    }

    HttpRequestBuilder& method(std::string m) {
        request_->method_ = std::move(m);
        return *this;   // คืน reference ของตัวเอง เพื่อให้ chain ต่อ (fluent interface)
    }

    HttpRequestBuilder& body(std::string b) {
        request_->body_ = std::move(b);
        return *this;
    }

    HttpRequestBuilder& timeout(int ms) {
        request_->timeoutMs_ = ms;
        return *this;
    }

    HttpRequestBuilder& noRedirects() {
        request_->followRedirects_ = false;
        return *this;
    }

    HttpRequestBuilder& skipSslVerification() {
        request_->verifySsl_ = false;
        return *this;
    }

    std::unique_ptr<HttpRequest> build() {
        return std::move(request_);   // ส่ง ownership ให้ผู้เรียก ผ่าน move ไม่ใช่ copy
    }

private:
    std::unique_ptr<HttpRequest> request_;
};

int main() {
    // อ่านง่ายกว่าเดิมมาก: รู้ทันทีว่าแต่ละค่าคืออะไรจากชื่อ method
    std::unique_ptr<HttpRequest> req =
        HttpRequestBuilder("https://api.example.com/users")
            .method("POST")
            .body("{\"name\":\"Nan\"}")
            .timeout(3000)
            .skipSslVerification()
            .build();

    req->describe();

    // ตัวอย่างที่สอง: ใช้ค่า default ล้วนๆ ไม่ต้องเรียก method ใดๆ เพิ่ม
    std::unique_ptr<HttpRequest> simpleGet =
        HttpRequestBuilder("https://api.example.com/health").build();
    simpleGet->describe();
}
```

ผลลัพธ์:

```
POST https://api.example.com/users (timeout=3000ms, redirects=true, verifySsl=false)
  body: {"name":"Nan"}
GET https://api.example.com/health (timeout=5000ms, redirects=true, verifySsl=true)
```

จุดที่ทำให้ code นี้เป็น **"Modern C++ Builder"**:

- ใช้ `std::unique_ptr<HttpRequest>` ถือ ownership ระหว่างประกอบร่าง แทน raw pointer —
  ถ้าเกิด exception ระหว่างเรียก method ใดๆ ก็ตาม memory จะถูกคืนอัตโนมัติ ไม่รั่วไหล
- `build()` คืนค่าด้วย `std::move(request_)` เพื่อโอน ownership ให้ผู้เรียกโดยไม่ copy —
  ทบทวนแนวคิด move semantics จาก Part 70
- แต่ละ setter method คืน `HttpRequestBuilder&` (reference ของตัวเอง) ทำให้เรียกต่อกัน
  เป็นสาย (chain) ได้ เรียกว่า **Fluent Interface** — อ่านคล้ายประโยคภาษาอังกฤษ

### Builder + Director: เมื่อมี "สูตรมาตรฐาน" ในการประกอบ

บางครั้งเรามี "ชุดค่าคงที่" ที่ใช้ประกอบซ้ำบ่อยๆ (เช่น "Gaming PC" ที่ต้องมี CPU/RAM/GPU
สเปกสูงเสมอ) GoF เสนอตัวช่วยเพิ่มเติมเรียกว่า **Director** — class ที่รู้ "ลำดับขั้นตอน"
มาตรฐานในการเรียก builder แต่ไม่รู้รายละเอียดของแต่ละ part เอง:

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

// ---------- Product ----------
class Computer {
public:
    void addPart(const std::string& part) { parts_.push_back(part); }

    void describe() const {
        std::cout << "สเปกคอมพิวเตอร์:\n";
        for (const auto& part : parts_) {
            std::cout << "  - " << part << '\n';
        }
    }

private:
    std::vector<std::string> parts_;
};

// ---------- Abstract Builder: กำหนดขั้นตอนย่อยที่ต้องมี ----------
class ComputerBuilder {
public:
    virtual ~ComputerBuilder() = default;
    virtual void buildCpu() = 0;
    virtual void buildRam() = 0;
    virtual void buildStorage() = 0;
    virtual void buildGpu() = 0;

    std::unique_ptr<Computer> release() { return std::move(computer_); }

protected:
    std::unique_ptr<Computer> computer_ = std::make_unique<Computer>();
};

class GamingComputerBuilder : public ComputerBuilder {
public:
    void buildCpu() override { computer_->addPart("CPU: Intel Core i9 (24 core)"); }
    void buildRam() override { computer_->addPart("RAM: 64GB DDR5"); }
    void buildStorage() override { computer_->addPart("Storage: 2TB NVMe SSD"); }
    void buildGpu() override { computer_->addPart("GPU: RTX 4090 24GB"); }
};

class OfficeComputerBuilder : public ComputerBuilder {
public:
    void buildCpu() override { computer_->addPart("CPU: Intel Core i5 (6 core)"); }
    void buildRam() override { computer_->addPart("RAM: 16GB DDR4"); }
    void buildStorage() override { computer_->addPart("Storage: 512GB SSD"); }
    void buildGpu() override { computer_->addPart("GPU: iGPU ในตัว (ไม่มีการ์ดจอแยก)"); }
};

// ---------- Director: รู้ "ลำดับขั้นตอน" มาตรฐานในการประกอบ แต่ไม่รู้รายละเอียดแต่ละ part ----------
class ComputerDirector {
public:
    std::unique_ptr<Computer> construct(ComputerBuilder& builder) {
        builder.buildCpu();
        builder.buildRam();
        builder.buildStorage();
        builder.buildGpu();
        return builder.release();
    }
};

int main() {
    ComputerDirector director;

    GamingComputerBuilder gamingBuilder;
    std::unique_ptr<Computer> gamingPc = director.construct(gamingBuilder);
    std::cout << "-- ชุด Gaming PC --\n";
    gamingPc->describe();

    OfficeComputerBuilder officeBuilder;
    std::unique_ptr<Computer> officePc = director.construct(officeBuilder);
    std::cout << "-- ชุด Office PC --\n";
    officePc->describe();
}
```

ผลลัพธ์:

```
-- ชุด Gaming PC --
สเปกคอมพิวเตอร์:
  - CPU: Intel Core i9 (24 core)
  - RAM: 64GB DDR5
  - Storage: 2TB NVMe SSD
  - GPU: RTX 4090 24GB
-- ชุด Office PC --
สเปกคอมพิวเตอร์:
  - CPU: Intel Core i5 (6 core)
  - RAM: 16GB DDR4
  - Storage: 512GB SSD
  - GPU: iGPU ในตัว (ไม่มีการ์ดจอแยก)
```

**เมื่อไหร่ใช้ Director**: ใช้เมื่อมี "สูตรมาตรฐาน" ที่ต้องประกอบซ้ำๆ เหมือนเดิมทุกครั้ง
(เช่น preset ของสินค้า) ถ้าการประกอบยืดหยุ่นมาก ไม่มีสูตรตายตัว (เหมือนตัวอย่าง
`HttpRequestBuilder` ก่อนหน้า ที่ผู้เรียกเลือกเองว่าจะเรียก method ไหนบ้าง) ก็ไม่จำเป็นต้องมี
Director เลย — Fluent Builder เพียวๆ ก็เพียงพอแล้ว ในโค้ด C++ สมัยใหม่ Fluent Builder แบบ
ไม่มี Director เป็นรูปแบบที่พบเห็นบ่อยกว่ามาก

---

## 97.6 Prototype Pattern: สร้าง Object ใหม่จากการ Clone (Step 774)

### ปัญหาที่ Prototype แก้

บางครั้งการสร้าง object ใหม่ **แพงกว่า** การ copy object ที่มีอยู่แล้วมาก เช่น object ที่ต้อง
โหลดข้อมูลจากไฟล์หรือฐานข้อมูลตอนสร้าง หรือบางครั้งเราแค่ต้องการ "สำเนา" ของ object ที่มี
state ปัจจุบันอยู่แล้ว (เช่น shape ที่ผู้ใช้วาดไว้ในโปรแกรมกราฟิก แล้วกด "Duplicate")
**Prototype Pattern** แก้ปัญหานี้โดยให้แต่ละ class รู้จัก **"โคลนตัวเอง"** ผ่านฟังก์ชัน
`clone()` แทนที่ client จะต้องรู้วิธีสร้าง object ชนิดนั้นตั้งแต่ต้น

```
┌─────────────────┐
│      Shape        │ (Prototype interface)
├─────────────────┤
│ +clone()* -> unique_ptr<Shape>│
│ +draw()*          │
└─────────────────┘
        △
   ┌────┴─────┐
┌────────┐ ┌───────────┐
│ Circle │ │ Rectangle │
├────────┤ ├───────────┤
│+clone()│ │+clone()   │  <- เรียก copy constructor ของตัวเอง (Circle(*this))
└────────┘ └───────────┘
```

```cpp
#include <iostream>
#include <map>
#include <memory>
#include <string>

// ---------- Prototype interface: ต้อง clone ตัวเองได้ ----------
class Shape {
public:
    virtual ~Shape() = default;
    virtual std::unique_ptr<Shape> clone() const = 0;   // หัวใจของ Prototype Pattern
    virtual void draw() const = 0;
};

class Circle : public Shape {
public:
    Circle(int x, int y, int radius) : x_(x), y_(y), radius_(radius) {}

    // copy constructor ธรรมดาของ C++ ทำงานร่วมกับ clone() ได้พอดี เพราะ Circle
    // ไม่มี resource พิเศษที่ต้อง deep copy เอง (สมาชิกเป็น int ล้วน)
    Circle(const Circle&) = default;

    std::unique_ptr<Shape> clone() const override {
        return std::make_unique<Circle>(*this);   // เรียก copy constructor ของ Circle เอง
    }

    void draw() const override {
        std::cout << "วงกลม ที่ (" << x_ << ", " << y_ << ") รัศมี " << radius_ << '\n';
    }

private:
    int x_, y_, radius_;
};

class Rectangle : public Shape {
public:
    Rectangle(int width, int height) : width_(width), height_(height) {}
    Rectangle(const Rectangle&) = default;

    std::unique_ptr<Shape> clone() const override {
        return std::make_unique<Rectangle>(*this);
    }

    void draw() const override {
        std::cout << "สี่เหลี่ยม ขนาด " << width_ << "x" << height_ << '\n';
    }

private:
    int width_, height_;
};

// ---------- Prototype Registry: เก็บ "ต้นแบบ" ไว้ล่วงหน้า แล้ว clone ตามต้องการ ----------
class ShapeRegistry {
public:
    void registerPrototype(const std::string& key, std::unique_ptr<Shape> prototype) {
        prototypes_[key] = std::move(prototype);
    }

    std::unique_ptr<Shape> create(const std::string& key) const {
        auto it = prototypes_.find(key);
        if (it == prototypes_.end()) {
            return nullptr;
        }
        return it->second->clone();   // สร้าง object ใหม่จากการ clone ต้นแบบ ไม่ใช่สร้างจาก 0
    }

private:
    std::map<std::string, std::unique_ptr<Shape>> prototypes_;
};

int main() {
    ShapeRegistry registry;
    registry.registerPrototype("small-circle", std::make_unique<Circle>(0, 0, 5));
    registry.registerPrototype("default-rect", std::make_unique<Rectangle>(10, 20));

    std::unique_ptr<Shape> c1 = registry.create("small-circle");
    std::unique_ptr<Shape> c2 = registry.create("small-circle");
    std::unique_ptr<Shape> r1 = registry.create("default-rect");

    c1->draw();
    c2->draw();
    r1->draw();

    std::cout << std::boolalpha
              << "c1 และ c2 เป็นคนละ object กัน: " << (c1.get() != c2.get()) << '\n';
}
```

ผลลัพธ์:

```
วงกลม ที่ (0, 0) รัศมี 5
วงกลม ที่ (0, 0) รัศมี 5
สี่เหลี่ยม ขนาด 10x20
c1 และ c2 เป็นคนละ object กัน: true
```

### ทำไม Prototype ต้องใช้ "Covariant Return Type" กับ unique_ptr

สังเกตว่า `clone()` ใน base class `Shape` คืน `std::unique_ptr<Shape>` และใน `Circle`,
`Rectangle` ก็คืน `std::unique_ptr<Shape>` เช่นกัน (ไม่ใช่ `std::unique_ptr<Circle>` ตรงๆ)
เพราะ **`unique_ptr<Derived>` ไม่สามารถแปลงเป็น `unique_ptr<Base>` แบบ covariant return
type ได้โดยตรงในการ override** (ต่างจาก raw pointer `Derived*` ที่ override เป็น `Base*`
ได้ตรงๆ เพราะ raw pointer รองรับ covariant return type ตั้งแต่ C++98) ในโค้ดข้างต้นเราคืน
`unique_ptr<Shape>` เหมือนกันทุก override ซึ่งเป็นวิธีที่ตรงไปตรงมาที่สุด — `make_unique<Circle>`
สร้าง `unique_ptr<Circle>` แล้ว compiler แปลงเป็น `unique_ptr<Shape>` ให้อัตโนมัติตอน
`return` เพราะ `Circle` สืบทอดจาก `Shape` (upcast ปลอดภัยเสมอ)

**ความเชื่อมโยงกับ Copy Constructor**: `clone()` ในทุก concrete class เรียก copy
constructor ของ class ตัวเอง (`Circle(*this)`) ผ่าน `std::make_unique<Circle>(*this)` — นี่คือ
เหตุผลที่ Prototype Pattern เชื่อมโยงโดยตรงกับแนวคิด **copy constructor** ที่เรียนมาตั้งแต่
Module C: ถ้า class ของเรามี resource พิเศษ (เช่น raw pointer ที่ต้อง deep copy, หรือ handle
ของไฟล์) เราต้อง implement copy constructor ให้ถูกต้องก่อน (ตาม **Rule of Three/Five** จาก
Module C-D) ไม่เช่นนั้น `clone()` ที่เรียก copy constructor แบบ default จะกลาย shallow copy
ที่ผิดพลาดทันที

---

## เปรียบเทียบ Creational Pattern ทั้ง 5 แบบ (Step 775)

| Pattern | แก้ปัญหาอะไร | ใช้เมื่อไหร่ | จุดเด่นในเชิง C++ |
|---|---|---|---|
| **Singleton** | ต้องการ object เดียวทั่วโปรแกรม + จุดเข้าถึงสากล | Logger, Config, Connection Pool | `static` local variable รับประกัน thread-safe ฟรีตั้งแต่ C++11 |
| **Factory Method** | แยก "การสร้าง product ชนิดเดียว" ออกจาก logic ที่ใช้ product | ต้องการให้ subclass เลือกชนิด product ที่สร้าง | คืนค่าเป็น `unique_ptr<Base>` ปลอดภัยจาก memory leak |
| **Abstract Factory** | สร้าง product หลายชนิดที่ต้อง "เข้าคู่กัน" เป็นตระกูล | ระบบ theme, ระบบ cross-platform UI | ส่ง factory object ผ่าน reference/pointer สลับ family ได้ตอน runtime |
| **Builder** | Constructor มีพารามิเตอร์ optional เยอะเกินไป | Object ที่มี config หลายส่วน เช่น HTTP Request, SQL Query | Fluent interface + `unique_ptr` + `std::move` ใน `build()` |
| **Prototype** | สร้าง object แพง หรืออยากได้สำเนาจาก state ปัจจุบัน | Editor ที่มีปุ่ม Duplicate, game object spawner | เชื่อมกับ copy constructor และ Rule of Three/Five โดยตรง |

### แผนภาพตัดสินใจอย่างง่าย

```
ต้องการ object เดียวทั่วโปรแกรม?
  └─ ใช่ ──▶ Singleton

ต้องการสร้าง object โดยไม่ให้ client รู้จัก concrete class?
  ├─ สร้างชนิดเดียว โดยให้ subclass เลือก ──▶ Factory Method
  └─ สร้างหลายชนิดที่ต้องเข้าคู่กัน (family) ──▶ Abstract Factory

Constructor มีพารามิเตอร์ optional เยอะจนอ่านยาก?
  └─ ใช่ ──▶ Builder

ต้องการสำเนาของ object ที่มีอยู่แล้ว (state ปัจจุบัน) แทนสร้างใหม่?
  └─ ใช่ ──▶ Prototype
```

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ Singleton แทน Global Variable โดยไม่คิดให้ดี** — Singleton ยังคงเป็น "state ที่ใครก็
   แก้ได้จากทุกที่ในโปรแกรม" เหมือน global variable ทุกประการ เพียงแค่ห่อด้วย class เท่านั้น
   การใช้ Singleton พร่ำเพรื่อทำให้โค้ดทดสอบยาก (unit test แต่ละ test case อาจเห็น state
   ที่ test อื่นทิ้งไว้) และทำให้ dependency ระหว่าง class มองไม่เห็นจาก signature ของฟังก์ชัน
   (เพราะไม่ได้ส่งผ่าน parameter) แนวทางที่ดีกว่าในระบบใหญ่คือ **Dependency Injection** —
   ส่ง object ที่ต้องการผ่าน constructor/parameter แทนการเรียก `getInstance()` ตรงๆ ทุกที่
2. **ลืมว่า Meyer's Singleton ทำลาย instance ตาม static destruction order ที่ไม่แน่นอน**
   — ถ้ามี Singleton สองตัวที่ destructor ของตัวหนึ่งต้องใช้อีกตัวหนึ่ง (เช่น Logger กับ
   FileManager) ลำดับการทำลาย static object ข้าม translation unit **ไม่ถูกกำหนดไว้ตาม
   มาตรฐาน** อาจทำให้ destructor เข้าถึง Singleton ที่ถูกทำลายไปแล้ว (**Static Initialization
   Order Fiasco** ที่เคยพูดถึงใน Part 53) วิธีแก้คือหลีกเลี่ยงการให้ Singleton พึ่งพากันเอง
   ใน destructor
3. **Factory ที่ใช้ raw pointer แล้วลืม `delete`** — ถ้าเขียน Factory Method แบบเก่าที่คืน
   `Shape*` ตรงๆ (ไม่ใช้ `unique_ptr`) ผู้เรียกต้องจำ `delete` เอง และถ้ามี exception เกิดขึ้น
   ระหว่างทางก่อนถึง `delete` จะเกิด memory leak ทันที — แก้ด้วยการคืน `unique_ptr`/`shared_ptr`
   เสมอตามตัวอย่างในบทนี้
4. **Builder ที่ไม่ป้องกัน `build()` ถูกเรียกซ้ำ** — ในตัวอย่างข้างต้น เมื่อ `build()` เรียก
   `std::move(request_)` แล้ว หาก builder object เดิมถูกเรียก `build()` ซ้ำอีกครั้ง จะได้
   `nullptr` กลับมาแทน (เพราะ `unique_ptr` ที่ถูก move ไปแล้วจะว่างเปล่า) ควรออกแบบให้ builder
   ใช้ครั้งเดียวทิ้ง หรือใส่ assertion/exception เตือนถ้ามีการเรียก `build()` ซ้ำโดยไม่ตั้งใจ
5. **Prototype ที่ลืม implement deep copy ให้ member ที่เป็น pointer/handle** — ถ้า concrete
   class มี raw pointer เป็น member (เช่น buffer ที่ alloc เอง) และปล่อยให้ `clone()` เรียก
   copy constructor แบบ default ที่ compiler generate ให้ จะได้ **shallow copy** ที่สอง object
   ชี้ไปที่ memory เดียวกัน พอ object ใดตัวหนึ่งถูกทำลายจะเกิด double-free หรือ dangling
   pointer ในอีกตัว — ต้องเขียน copy constructor เอง (Rule of Three/Five) ให้ deep copy
   resource เหล่านั้นก่อนเสมอ
6. **สร้าง Abstract Factory ทั้งที่มี product แค่ชนิดเดียว** — ถ้าระบบมี concrete product แค่
   ตัวเดียวและไม่มีแผนจะเพิ่ม "ตระกูล" ใหม่ การสร้าง `GUIFactory` abstract พร้อม interface
   เต็มรูปแบบเป็นการเพิ่มความซับซ้อนโดยไม่จำเป็น (ดูหัวข้อ over-engineering ใน Part 98)

---

## แบบฝึกหัดท้ายบท

1. เขียน Meyer's Singleton ชื่อ `IdGenerator` ที่มี method `nextId()` คืนค่า `int` ที่เพิ่มขึ้น
   ทีละ 1 ทุกครั้งที่ถูกเรียก (เริ่มจาก 1) ทดสอบเรียก `nextId()` 3 ครั้งติดกันแล้วพิมพ์ผลลัพธ์
2. ขยายตัวอย่าง Factory Method (`Application`/`Document`) ในบทเรียน โดยเพิ่มชนิดเอกสารใหม่
   ชื่อ `PresentationDocument` และ `PresentationApplication` โดยไม่ต้องแก้โค้ดเดิมของ
   `Application`, `TextApplication`, `SpreadsheetApplication` เลยแม้แต่บรรทัดเดียว
3. ขยายตัวอย่าง Abstract Factory (`GUIFactory`) ในบทเรียน โดยเพิ่ม product ชนิดที่สาม
   `Slider` (พร้อม `LightSlider`, `DarkSlider`) และปรับ `GUIFactory`, `LightFactory`,
   `DarkFactory` ให้รองรับ
4. เขียน Builder Pattern สำหรับ class `PizzaOrder` ที่มีฟิลด์ `size` (string เช่น "M"),
   `toppings` (`std::vector<std::string>`), `extraCheese` (bool), `deliveryAddress` (string)
   โดยใช้ fluent interface แบบเดียวกับ `HttpRequestBuilder`
5. เขียน Prototype Pattern สำหรับ class `Enemy` ในเกม ที่มี `health`, `attackPower`, และ
   `name` โดยสร้าง `EnemyRegistry` ที่ลงทะเบียนศัตรู 2 ชนิด (`"goblin"`, `"dragon"`) แล้ว
   ทดลอง spawn ศัตรูชนิด `"goblin"` 3 ตัวจาก prototype เดียวกัน
6. อธิบายด้วยคำพูดตัวเอง (ไม่ต้องเขียนโค้ด) ว่าทำไม Builder Pattern ถึงสำคัญกว่าในภาษา C++
   เมื่อเทียบกับภาษาอย่าง Python หรือ Kotlin ที่มี named/default argument ในตัว

### แนวทางเฉลยข้อ 1

```cpp
#include <iostream>

class IdGenerator {
public:
    IdGenerator(const IdGenerator&) = delete;
    IdGenerator& operator=(const IdGenerator&) = delete;

    static IdGenerator& instance() {
        static IdGenerator generator;
        return generator;
    }

    int nextId() { return ++counter_; }

private:
    IdGenerator() = default;
    int counter_ = 0;
};

int main() {
    std::cout << IdGenerator::instance().nextId() << '\n';
    std::cout << IdGenerator::instance().nextId() << '\n';
    std::cout << IdGenerator::instance().nextId() << '\n';
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 id_generator.cpp -o id_generator
./id_generator
```

ผลลัพธ์:

```
1
2
3
```

### แนวทางเฉลยข้อ 4

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <utility>
#include <vector>

class PizzaOrder {
public:
    void describe() const {
        std::cout << "พิซซ่าไซซ์ " << size_ << " ส่งไปที่ " << address_;
        std::cout << (extraCheese_ ? " (เพิ่มชีส)" : "") << '\n';
        if (!toppings_.empty()) {
            std::cout << "  หน้า: ";
            for (const auto& t : toppings_) std::cout << t << " ";
            std::cout << '\n';
        }
    }

    friend class PizzaOrderBuilder;

private:
    std::string size_ = "M";
    std::vector<std::string> toppings_;
    bool extraCheese_ = false;
    std::string address_;
};

class PizzaOrderBuilder {
public:
    explicit PizzaOrderBuilder(std::string address) {
        order_ = std::make_unique<PizzaOrder>();
        order_->address_ = std::move(address);
    }

    PizzaOrderBuilder& size(std::string s) {
        order_->size_ = std::move(s);
        return *this;
    }

    PizzaOrderBuilder& addTopping(std::string topping) {
        order_->toppings_.push_back(std::move(topping));
        return *this;
    }

    PizzaOrderBuilder& extraCheese() {
        order_->extraCheese_ = true;
        return *this;
    }

    std::unique_ptr<PizzaOrder> build() { return std::move(order_); }

private:
    std::unique_ptr<PizzaOrder> order_;
};

int main() {
    std::unique_ptr<PizzaOrder> order =
        PizzaOrderBuilder("123 ถนนสุขุมวิท")
            .size("L")
            .addTopping("เห็ด")
            .addTopping("ไก่")
            .extraCheese()
            .build();

    order->describe();
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 pizza_builder.cpp -o pizza_builder
./pizza_builder
```

ผลลัพธ์:

```
พิซซ่าไซซ์ L ส่งไปที่ 123 ถนนสุขุมวิท (เพิ่มชีส)
  หน้า: เห็ด ไก่
```

(ข้อ 2, 3, 5 และ 6 ให้ผู้เรียนลองทำเองตามแนวทางของตัวอย่างในบทเรียน แล้วเทียบโครงสร้างกับ
ตัวอย่างที่ให้ไว้ — สิ่งสำคัญคือ Factory Method/Abstract Factory ในข้อ 2-3 ต้องไม่แก้โค้ด
class เดิมเลยแม้แต่บรรทัดเดียว มีแต่การเพิ่ม class ใหม่เท่านั้น ซึ่งคือหัวใจของ Open/Closed
Principle ที่ pattern เหล่านี้ช่วยให้ทำได้จริง)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Design Pattern คือ "คำตอบที่มีชื่อเรียก" สำหรับปัญหาการออกแบบที่เกิดซ้ำๆ ไม่ใช่
  โค้ดสำเร็จรูป และรู้จัก GoF กับ 3 หมวดหมู่หลัก (Creational, Structural, Behavioral)
- ทบทวนและต่อยอด **Singleton** ให้ลึกขึ้น: เข้าใจว่าทำไม Meyer's Singleton ปลอดภัยตั้งแต่
  C++11 ทำไม double-checked locking แบบเก่าเป็นอันตราย และเมื่อไหร่ควรใช้ `std::call_once`
- ใช้ **Factory Method** แยกการสร้าง product ออกจากโค้ดที่ใช้ product ผ่าน virtual function
- ใช้ **Abstract Factory** การันตีว่า product หลายชนิดที่เกี่ยวข้องกันจะถูกสร้างเป็น "ตระกูล"
  เดียวกันเสมอ
- แก้ปัญหา telescoping constructor ด้วย **Builder Pattern** และ fluent interface พร้อม
  เข้าใจบทบาทเสริมของ Director
- ใช้ **Prototype Pattern** สร้าง object ใหม่จากการ clone และเชื่อมโยงกับ copy constructor
  และ Rule of Three/Five จาก Module C-D
- ทุกตัวอย่างใช้ `std::unique_ptr`/`std::make_unique` แทน raw pointer ตามแนวทาง Modern C++
  ที่เรียนมาตั้งแต่ Part 67

ใน **Part 98** เราจะเรียนต่อในหมวด **Structural Pattern** (Adapter, Decorator, Facade,
Composite) และ **Behavioral Pattern** (Observer, Strategy, Command, Iterator) ซึ่ง Observer
จะสำคัญมากเป็นพิเศษเมื่อเราเริ่มเข้าสู่ **Module I: Web Development** ที่เต็มไปด้วยระบบ
event-driven

**ต่อไป:** [Part 98 — Design Pattern ใน C++ (ตอนที่ 2): Structural และ Behavioral Patterns](./part-098-design-patterns-2.md)
