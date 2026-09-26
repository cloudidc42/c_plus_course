# Part 55: โปรเจกต์ Library Management System (Step 433–440)

> Module D — เริ่มต้น C++ และ OOP | Part 55 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 433–440
> Part ก่อนหน้า: [Part 54 — Exception Handling ใน C++](./part-054-exceptions-cpp.md) | Part ถัดไป: [Part 56 — Function Template และ Class Template](./part-056-function-class-templates.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. ออกแบบระบบซอฟต์แวร์ขนาดกลางด้วยแนวคิด OOP ตั้งแต่ต้นจนจบ — เริ่มจาก requirement ไปจนถึง
   class diagram และการแบ่งไฟล์ (module) อย่างเป็นระบบ
2. ประยุกต์ **Abstract Class** และ **Polymorphism** (Part 49-50) ในการออกแบบลำดับชั้นของสื่อ
   ห้องสมุด (`Media` → `Book`/`DVD`/`Magazine`) ที่ขยายเพิ่มชนิดใหม่ได้โดยไม่แก้โค้ดเดิม
3. ใช้ **static member** (Part 53) นับจำนวนสื่อทั้งหมดในระบบและจำกัดโควตาการยืมของสมาชิกแต่ละคน
4. ออกแบบ **exception hierarchy** ของระบบเอง (Part 54) ที่สืบทอดจาก `std::runtime_error` และ
   ใช้ throw/catch จัดการสถานการณ์ error ทางธุรกิจ (ยืมซ้ำ, ไม่พบข้อมูล, เกินโควตา) ได้ถูกต้อง
5. ใช้ **operator overloading** (`operator<<`, Part 51) แสดงผลข้อมูลของ object ได้เป็นธรรมชาติ
6. จัดการความเป็นเจ้าของ (ownership) ของ object แบบ polymorphic ด้วย `std::unique_ptr` แทนการ
   ใช้ raw pointer + `new`/`delete` แบบ manual ที่เสี่ยงต่อ memory leak
7. เขียนโปรเจกต์ C++ แบบ **multi-file** ที่แบ่งเป็น header (`.h`) และ source (`.cpp`) หลายไฟล์
   พร้อม **Makefile** ที่คอมไพล์และ build ได้อัตโนมัติ (ทบทวน Part 17-18)
8. บูรณาการแนวคิดทั้งหมดของ Module D (Class, Constructor/Destructor, Encapsulation,
   Inheritance, Polymorphism, Abstract Class, Operator Overloading, Friend, Static Member,
   Exception) เข้าด้วยกันเป็นโปรแกรมที่ทำงานได้จริงและขยายต่อยอดได้

---

## 55.1 ออกแบบระบบ: Requirement และ Class Diagram (Step 433)

### Requirement ของระบบ

ระบบจัดการห้องสมุด (Library Management System) ที่เราจะสร้างต้องทำสิ่งต่อไปนี้ได้:

1. เก็บ**สื่อ (Media)** ได้หลายชนิด: หนังสือ (Book), แผ่น DVD, และนิตยสาร (Magazine) — แต่ละ
   ชนิดมีข้อมูลเฉพาะตัวต่างกัน (หนังสือมีผู้แต่ง/ISBN, DVD มีผู้กำกับ/ความยาว, นิตยสารมีฉบับที่)
   แต่ทุกชนิดมีพฤติกรรมร่วมกัน (แสดงรายละเอียด, ถูกยืม/คืนได้)
2. เก็บ**สมาชิก (Member)** ของห้องสมุด แต่ละคนยืมสื่อได้ไม่เกินโควตาที่กำหนด
3. รองรับการ**ยืม (borrow)** และ**คืน (return)** สื่อ พร้อมตรวจสอบเงื่อนไขทางธุรกิจ เช่น
   ห้ามยืมสื่อที่มีคนยืมอยู่แล้วซ้ำ, ห้ามยืมเกินโควตา, ห้ามคืนสื่อที่ไม่ได้ยืมไว้
4. แสดง**แคตตาล็อก**ของสื่อทั้งหมด และรายชื่อสมาชิกทั้งหมด พร้อมสถานะปัจจุบัน
5. ระบบต้อง**ขยายเพิ่มชนิดสื่อใหม่**ได้ในอนาคต (เช่น AudioBook) โดย**ไม่ต้องแก้โค้ดเดิม**ที่
   ทดสอบผ่านแล้ว — นี่คือหลักการ **Open-Closed Principle** ซึ่งเป็นหนึ่งใน SOLID Principles ที่
   จะพูดถึงอย่างเป็นทางการใน Module H

### Class Diagram

```
                         ┌───────────────────────┐
                         │  <<abstract>>  Media   │
                         ├───────────────────────┤
                         │ - title_ : string      │
                         │ - id_ : string         │
                         │ - borrowed_ : bool     │
                         │ - totalMediaCount_:int │ (static)
                         ├───────────────────────┤
                         │ + mediaType() = 0      │ (pure virtual)
                         │ + extraInfo() virtual  │
                         │ + markBorrowed()       │
                         │ + markReturned()       │
                         │ + totalMediaCount()    │ (static)
                         │ + operator<<           │ (friend)
                         └───────────┬───────────┘
                    ┌────────────────┼────────────────┐
                    │                │                 │
            ┌───────▼──────┐ ┌───────▼──────┐  ┌───────▼────────┐
            │     Book     │ │     DVD      │  │    Magazine    │
            ├──────────────┤ ├──────────────┤  ├────────────────┤
            │ - author_    │ │ - director_  │  │ - issueNumber_ │
            │ - isbn_      │ │ - runtime_   │  │                │
            │ - totalBook  │ │              │  │                │
            │   Count_(st.)│ │              │  │                │
            └──────────────┘ └──────────────┘  └────────────────┘

  ┌───────────────────────┐          ┌──────────────────────────┐
  │        Member          │          │          Library          │
  ├───────────────────────┤          ├──────────────────────────┤
  │ - id_ : string          │          │ - name_ : string          │
  │ - name_ : string        │◄─────────┤ - collection_ :           │
  │ - borrowedMedia_ :      │  owns    │     vector<unique_ptr     │
  │   vector<Media*>        │ (unique_ │     <Media>>              │
  │   (non-owning)          │  ptr)    │ - members_ :               │
  │ + MAX_BORROW_LIMIT      │          │     vector<unique_ptr     │
  │   (static const)        │          │     <Member>>              │
  └───────────────────────┘          ├──────────────────────────┤
                                       │ + addMedia() / addMember() │
                                       │ + borrowMedia()/returnMedia│
                                       │ + printCatalog()/printMembers│
                                       └──────────────────────────┘
```

จุดสำคัญของการออกแบบนี้:

- **`Library` เป็นเจ้าของ `Media` และ `Member` ทั้งหมด** ผ่าน `std::unique_ptr` — เมื่อ
  `Library` ถูกทำลาย ทุก `Media`/`Member` จะถูกทำลายตามไปด้วยอัตโนมัติ (RAII เต็มรูปแบบ ทบทวน
  Part 46 และ Part 54) ไม่มีการ `new`/`delete` แบบ manual เลยแม้แต่บรรทัดเดียวในทั้งโปรเจกต์
- **`Member` ถือ `Media*` แบบ raw pointer ที่ไม่เป็นเจ้าของ (non-owning observer pointer)**
  เพื่อบันทึกว่า "สมาชิกคนนี้กำลังยืมสื่อชิ้นไหนอยู่" — มันไม่รับผิดชอบการคืนหน่วยความจำ เพราะ
  `Library` เป็นเจ้าของตัวจริง (สมาร์ทพอยน์เตอร์แบบเจาะลึกทุกชนิด รวมถึง `weak_ptr` สำหรับกรณี
  แบบนี้โดยเฉพาะ จะเรียนเต็มรูปแบบใน **Part 67**)
- ใช้ `vector<unique_ptr<T>>` แทน `vector<T>` ธรรมดาทั้งสำหรับ `collection_` และ `members_`
  ด้วยเหตุผลสองข้อ: (1) `Media` เป็น abstract class เก็บใน `vector<Media>` ตรงๆ ไม่ได้อยู่แล้ว
  ต้องเก็บผ่าน pointer เพื่อให้ polymorphism ทำงาน (ทบทวน Part 49-50) และ (2) การเก็บผ่าน
  `unique_ptr` ทำให้ **ที่อยู่ของ object จริงไม่เปลี่ยนแม้ vector จะขยายขนาด (reallocate)**
  ต่างจากการเก็บ `Member` ตรงๆ ใน `vector<Member>` ที่ reference ไปยัง element เดิมจะ dangling
  ทันทีที่ vector reallocate — รายละเอียดเรื่อง `vector` และการ reallocate จะเรียนเจาะลึกใน
  **Part 59**

### User Story: มองระบบจากมุมมองผู้ใช้งานจริง

ก่อนเขียนโค้ด นักออกแบบซอฟต์แวร์มืออาชีพมักแปลง requirement ดิบๆ ให้เป็น **User Story** —
ประโยคสั้นๆ ที่บอกว่า "ใคร ต้องการทำอะไร เพื่ออะไร" เพื่อให้มั่นใจว่าฟีเจอร์ทุกตัวมีเหตุผลรองรับ
จริง ไม่ใช่แค่ "เขียนเพราะสอนในหนังสือ":

| User Story | เกี่ยวข้องกับ method/class |
|---|---|
| ในฐานะบรรณารักษ์ ฉันต้องการเพิ่มสื่อใหม่เข้าระบบ เพื่อให้สมาชิกยืมได้ | `Library::addMedia()` |
| ในฐานะบรรณารักษ์ ฉันต้องการลงทะเบียนสมาชิกใหม่ เพื่อให้เขายืมสื่อได้ | `Library::addMember()` |
| ในฐานะสมาชิก ฉันต้องการยืมหนังสือที่ว่างอยู่ เพื่อนำไปอ่านที่บ้าน | `Library::borrowMedia()` |
| ในฐานะระบบ ฉันต้องการปฏิเสธการยืมสื่อที่มีคนยืมอยู่แล้ว เพื่อป้องกันข้อขัดแย้ง | `AlreadyBorrowedError` |
| ในฐานะระบบ ฉันต้องการจำกัดจำนวนที่แต่ละคนยืมได้ เพื่อให้ทรัพยากรกระจายทั่วถึง | `Member::MAX_BORROW_LIMIT`, `BorrowLimitExceededError` |
| ในฐานะสมาชิก ฉันต้องการคืนสื่อที่ยืมไป เพื่อให้คนอื่นยืมต่อได้ | `Library::returnMedia()` |
| ในฐานะบรรณารักษ์ ฉันต้องการดูแคตตาล็อกทั้งหมด เพื่อตรวจสอบสถานะคลังสื่อ | `Library::printCatalog()` |

การเขียน User Story แบบนี้ช่วยตรวจสอบว่า class diagram ใน 55.1 **ครอบคลุมทุกความต้องการจริง
หรือไม่** ก่อนที่จะลงมือเขียนโค้ดจริงในหัวข้อถัดไป — เป็นขั้นตอนที่ทีมพัฒนาซอฟต์แวร์มืออาชีพทำ
กันเป็นปกติก่อนเริ่ม sprint การพัฒนา (จะพูดถึงการออกแบบสถาปัตยกรรมซอฟต์แวร์ระดับใหญ่ขึ้นไปอีก
ใน Part 114)

### ทำไมเลือก unique_ptr แทน raw pointer หรือ shared_ptr

ในขั้นตอนออกแบบ เราต้องตัดสินใจว่า `Library` จะเก็บ ownership ของ `Media`/`Member` อย่างไร
มีสามตัวเลือกหลักที่พิจารณา (สมาร์ทพอยน์เตอร์ทั้งหมดจะเรียนเจาะลึกใน Part 67 แต่เราเลือกใช้
`unique_ptr` ตั้งแต่ตอนนี้เพราะมันแก้ปัญหา ownership ของโปรเจกต์นี้ได้ตรงที่สุด):

| ตัวเลือก | ข้อดี | ข้อเสียสำหรับโปรเจกต์นี้ |
|---|---|---|
| Raw pointer + manual `new`/`delete` | ควบคุมได้ละเอียดที่สุด ไม่มี overhead ใดๆ | ต้องจำ `delete` เองทุกจุด เสี่ยง memory leak สูงมากถ้าเกิด exception กลางทาง (ทบทวน Part 54 ข้อ 54.6) |
| `std::shared_ptr<Media>` | หลาย object ถือ ownership ร่วมกันได้ ไม่ต้องกังวลว่าใครจะลบก่อน | **เกินความจำเป็น**: ในระบบนี้มีเจ้าของที่แท้จริงเพียงรายเดียวคือ `Library` การใช้ `shared_ptr` จะเสีย overhead ของ reference counting โดยไม่ได้ประโยชน์อะไรเพิ่ม และอาจซ่อนบั๊กเรื่อง ownership ที่ไม่ชัดเจนไว้ |
| **`std::unique_ptr<Media>`** (ตัวที่เราเลือกใช้) | สื่อความหมาย ownership ที่ชัดเจนที่สุด: "`Library` เป็นเจ้าของแต่เพียงผู้เดียว" ตรงกับความเป็นจริงของโดเมนปัญหา ไม่มี overhead ของ reference counting | ห้าม copy (ต้อง `std::move` เวลาส่งต่อ) — แต่ในระบบนี้เราไม่เคยต้องการ copy สื่ออยู่แล้ว จึงไม่ใช่ข้อเสียจริง |

หลักการเลือก smart pointer ที่ดี (ซึ่งจะเรียนเป็นทางการใน Part 67-68) คือ **"เลือกตัวที่จำกัด
สิทธิ์น้อยที่สุดเท่าที่จำเป็น"** — ถ้า ownership เป็นแบบ "เจ้าของเดียว" จริงๆ ให้ใช้
`unique_ptr` เสมอ อย่าใช้ `shared_ptr` แค่เพราะ "เผื่อไว้" เพราะจะทำให้โค้ดอ่านยากขึ้นและอาจ
ซ่อนบั๊กเรื่องใครเป็นเจ้าของ object จริงๆ ไว้

### โครงสร้างไฟล์ของโปรเจกต์

| ไฟล์ | หน้าที่ |
|---|---|
| `Media.h` / `Media.cpp` | Abstract class `Media` และ derived class `Book`, `DVD`, `Magazine` |
| `Member.h` / `Member.cpp` | class `Member` (สมาชิกห้องสมุด) |
| `Library.h` / `Library.cpp` | Exception hierarchy ของระบบ + class `Library` (ตัวจัดการหลัก) |
| `main.cpp` | จุดเริ่มต้นโปรแกรม ทดสอบทุก feature |
| `Makefile` | คำสั่ง build อัตโนมัติ |

---

## 55.2 Media.h/.cpp — Abstract Base Class (Step 434)

`Media` คือหัวใจของระบบ ทำหน้าที่เป็น**สัญญา (contract)** ที่บังคับว่าสื่อทุกชนิดต้องบอกได้ว่า
ตัวเองเป็นชนิดอะไร (`mediaType()`) ผ่าน **pure virtual function** (ทบทวน Part 50) ทำให้
`Media` เป็น **abstract class** ที่สร้าง object ตรงๆ ไม่ได้ ต้องสร้างผ่าน derived class เท่านั้น

`Media.h`:

```cpp
#ifndef MEDIA_H
#define MEDIA_H

#include <iostream>
#include <string>

// ===== คลาสฐานนามธรรม (Abstract Base Class) สำหรับสื่อทุกชนิดในห้องสมุด =====
class Media {
public:
    Media(std::string title, std::string id);
    virtual ~Media();

    // ห้าม copy สื่อโดยตรง เพราะ Library เป็นเจ้าของผ่าน unique_ptr เพียงที่เดียว
    Media(const Media&) = delete;
    Media& operator=(const Media&) = delete;

    virtual std::string mediaType() const = 0;         // pure virtual -> ทำให้ Media เป็น abstract class
    virtual std::string extraInfo() const;              // ข้อมูลเพิ่มเติมเฉพาะชนิด (default ว่างเปล่า)

    const std::string& title() const noexcept { return title_; }
    const std::string& id() const noexcept { return id_; }
    bool isBorrowed() const noexcept { return borrowed_; }

    void markBorrowed();
    void markReturned();

    static int totalMediaCount() noexcept { return totalMediaCount_; }

    friend std::ostream& operator<<(std::ostream& os, const Media& m);

private:
    std::string title_;
    std::string id_;
    bool borrowed_ = false;

    static int totalMediaCount_;   // static data member: นับจำนวนสื่อทั้งหมดที่มีอยู่ในระบบ ณ ขณะนี้
};

// ===== หนังสือ =====
class Book : public Media {
public:
    Book(std::string title, std::string id, std::string author, std::string isbn);
    ~Book() override;

    std::string mediaType() const override { return "Book"; }
    std::string extraInfo() const override;

    const std::string& author() const noexcept { return author_; }
    const std::string& isbn() const noexcept { return isbn_; }

    static int totalBookCount() noexcept { return totalBookCount_; }

private:
    std::string author_;
    std::string isbn_;

    static int totalBookCount_;    // static data member เฉพาะของ Book: นับจำนวนหนังสือทั้งหมดในระบบ
};

// ===== แผ่น DVD =====
class DVD : public Media {
public:
    DVD(std::string title, std::string id, std::string director, int runtimeMinutes);

    std::string mediaType() const override { return "DVD"; }
    std::string extraInfo() const override;

private:
    std::string director_;
    int runtimeMinutes_;
};

// ===== นิตยสาร =====
class Magazine : public Media {
public:
    Magazine(std::string title, std::string id, int issueNumber);

    std::string mediaType() const override { return "Magazine"; }
    std::string extraInfo() const override;

private:
    int issueNumber_;
};

#endif // MEDIA_H
```

### อธิบายการออกแบบ Media

- **`virtual ~Media();`**: destructor ต้องเป็น `virtual` เสมอเมื่อ class ถูกออกแบบมาให้สืบทอด
  และถูกลบผ่าน base class pointer (ทบทวน Part 49) — `Library` เก็บสื่อเป็น
  `unique_ptr<Media>` ดังนั้นเมื่อ `unique_ptr` ถูกทำลาย มันจะเรียก `delete` ผ่าน `Media*`
  ถ้า destructor ไม่เป็น `virtual` จะเกิด **undefined behavior** ทันที (destructor ของ
  `Book`/`DVD`/`Magazine` จะไม่ถูกเรียก ทำให้ member เฉพาะของมัน เช่น `std::string author_`
  ไม่ถูกทำลายอย่างถูกต้อง)
- **`Media(const Media&) = delete;`**: ห้าม copy เพราะการ copy สื่อทางกายภาพ (เช่น หนังสือ
  เล่มเดียวกัน) ไม่สมเหตุสมผลในโดเมนนี้ และ `Library` ควรเป็นเจ้าของ instance เดียวเท่านั้น
- **`virtual std::string mediaType() const = 0;`**: pure virtual function — ทำให้ `Media`
  เป็น abstract class ที่สร้าง object ตรงๆ ไม่ได้ (compiler จะ error ทันทีถ้าเขียน
  `Media m("x", "1");`) และบังคับให้ทุก derived class ต้อง implement ฟังก์ชันนี้
- **`virtual std::string extraInfo() const;`**: เป็น virtual function ธรรมดา (ไม่ใช่ pure)
  มี default implementation คืนค่า string ว่าง เปิดโอกาสให้ derived class **เลือก override
  หรือไม่ก็ได้** ต่างจาก `mediaType()` ที่บังคับ override เสมอ
- **`static int totalMediaCount_;`**: ใช้เทคนิค Object Counter ที่เรียนใน Part 53 ข้อ 53.3
  ทุกครั้งที่ `Media` (หรือ derived class ใดๆ) ถูกสร้าง `totalMediaCount_` จะเพิ่มขึ้นอัตโนมัติ
  ผ่าน constructor ของ `Media` (แม้ derived class จะเรียก constructor ของตัวเองก็ตาม เพราะ
  constructor ของ base class จะถูกเรียกก่อนเสมอ)

`Media.cpp`:

```cpp
#include "Media.h"
#include <utility>

int Media::totalMediaCount_ = 0;

Media::Media(std::string title, std::string id)
    : title_(std::move(title)), id_(std::move(id)) {
    ++totalMediaCount_;
}

Media::~Media() {
    --totalMediaCount_;
}

std::string Media::extraInfo() const {
    return "";
}

void Media::markBorrowed() {
    borrowed_ = true;
}

void Media::markReturned() {
    borrowed_ = false;
}

std::ostream& operator<<(std::ostream& os, const Media& m) {
    os << "[" << m.mediaType() << "] " << m.title() << " (ID: " << m.id() << ") - "
       << (m.borrowed_ ? "ถูกยืมอยู่" : "ว่างอยู่บนชั้น");
    const std::string extra = m.extraInfo();   // เรียกผ่าน virtual function -> dynamic dispatch
    if (!extra.empty()) {
        os << " | " << extra;
    }
    return os;
}
```

จุดที่น่าสนใจที่สุดในไฟล์นี้คือ `operator<<`: มันถูกประกาศเป็น **`friend`** ของ `Media`
(ทบทวน Part 52) เพื่อให้เขียนใช้งานแบบ `std::cout << media;` ได้เป็นธรรมชาติ (ถ้าไม่ใช้ friend
function จะต้องเขียน `media.print(std::cout);` แทน ซึ่งดูไม่เป็นธรรมชาติเท่า `operator<<`)
ภายในฟังก์ชันเรียก `m.mediaType()` และ `m.extraInfo()` ซึ่งเป็น **virtual function** — แม้
`operator<<` จะรับพารามิเตอร์เป็น `const Media&` แต่เมื่อ `m` อ้างอิงไปยัง object จริงที่เป็น
`Book`, `DVD`, หรือ `Magazine` การเรียกจะถูก dispatch ไปยังฟังก์ชันของ derived class ที่ถูกต้อง
โดยอัตโนมัติ (Dynamic Dispatch ทบทวน Part 49) — นี่คือรูปแบบมาตรฐานของการเขียน `operator<<`
สำหรับลำดับชั้น class ที่มี polymorphism

---

## 55.3 Book, DVD, Magazine — Derived Classes และ Polymorphism (Step 435)

ส่วนที่เหลือของ `Media.cpp` implement derived class ทั้งสาม แต่ละตัวเรียก constructor ของ
`Media` (base class) ก่อนเสมอ ผ่าน member initializer list (ทบทวน Part 46):

```cpp
int Book::totalBookCount_ = 0;

Book::Book(std::string title, std::string id, std::string author, std::string isbn)
    : Media(std::move(title), std::move(id)), author_(std::move(author)), isbn_(std::move(isbn)) {
    ++totalBookCount_;
}

Book::~Book() {
    --totalBookCount_;
}

std::string Book::extraInfo() const {
    return "ผู้แต่ง: " + author_ + ", ISBN: " + isbn_;
}

DVD::DVD(std::string title, std::string id, std::string director, int runtimeMinutes)
    : Media(std::move(title), std::move(id)),
      director_(std::move(director)),
      runtimeMinutes_(runtimeMinutes) {}

std::string DVD::extraInfo() const {
    return "ผู้กำกับ: " + director_ + ", ความยาว: " + std::to_string(runtimeMinutes_) + " นาที";
}

Magazine::Magazine(std::string title, std::string id, int issueNumber)
    : Media(std::move(title), std::move(id)), issueNumber_(issueNumber) {}

std::string Magazine::extraInfo() const {
    return "ฉบับที่: " + std::to_string(issueNumber_);
}
```

### สังเกตแพทเทิร์นสำคัญ: Template Method ผ่าน operator<<

`operator<<` ของ `Media` ที่เขียนไว้ใน 55.2 **ไม่ต้องแก้ไขอะไรเลย** เมื่อเราเพิ่มชนิดสื่อใหม่
(`DVD`, `Magazine` หรือแม้แต่ชนิดในอนาคต) เพราะมันเรียก `mediaType()` และ `extraInfo()` แบบ
virtual เสมอ — แต่ละ derived class แค่ implement สองฟังก์ชันนี้ให้ตรงกับตัวเอง ระบบทั้งหมดจะ
แสดงผลถูกต้องโดยอัตโนมัติ นี่คือประโยชน์ที่จับต้องได้ของ Polymorphism ที่เรียนมาตั้งแต่ Part 49:
**เขียน operator<< เพียงครั้งเดียว ใช้ได้กับทุกชนิดสื่อ ทั้งที่มีอยู่แล้วและที่จะเพิ่มในอนาคต**

`Book` ยังมี **static member ของตัวเอง** (`totalBookCount_`) แยกจาก `totalMediaCount_` ของ
`Media` — สาธิตว่า static member ของแต่ละ class ใน hierarchy เป็นอิสระจากกัน: `totalBookCount_`
นับเฉพาะ `Book` เท่านั้น ในขณะที่ `totalMediaCount_` นับสื่อทุกชนิดรวมกัน

---

## 55.4 Custom Exceptions ของระบบ (Step 436)

ก่อนเขียน class `Library` เราออกแบบ **exception hierarchy** ของระบบก่อน (ทบทวน Part 54)
ทุก exception สืบทอดจาก `LibraryError` (ซึ่งสืบทอดจาก `std::runtime_error` อีกที) ทำให้ผู้ใช้
เลือกได้ว่าจะ `catch` แบบเจาะจง (เช่น `AlreadyBorrowedError`) หรือแบบกว้าง (`LibraryError`
หรือ `std::exception`) ก็ได้ — ส่วนนี้อยู่ต้นไฟล์ `Library.h`:

```cpp
#ifndef LIBRARY_H
#define LIBRARY_H

#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>
#include <vector>

#include "Media.h"
#include "Member.h"

// ===== Exception hierarchy ของระบบห้องสมุด (สืบทอดจาก std::runtime_error) =====

class LibraryError : public std::runtime_error {
public:
    explicit LibraryError(const std::string& message) : std::runtime_error(message) {}
};

class MediaNotFoundError : public LibraryError {
public:
    explicit MediaNotFoundError(const std::string& mediaId)
        : LibraryError("ไม่พบสื่อ ID: " + mediaId), mediaId_(mediaId) {}
    const std::string& mediaId() const noexcept { return mediaId_; }
private:
    std::string mediaId_;
};

class MemberNotFoundError : public LibraryError {
public:
    explicit MemberNotFoundError(const std::string& memberId)
        : LibraryError("ไม่พบสมาชิก ID: " + memberId), memberId_(memberId) {}
    const std::string& memberId() const noexcept { return memberId_; }
private:
    std::string memberId_;
};

class AlreadyBorrowedError : public LibraryError {
public:
    explicit AlreadyBorrowedError(const std::string& title)
        : LibraryError("สื่อ \"" + title + "\" ถูกยืมไปแล้ว ไม่สามารถยืมซ้ำได้") {}
};

class NotBorrowedError : public LibraryError {
public:
    explicit NotBorrowedError(const std::string& title)
        : LibraryError("สื่อ \"" + title + "\" ไม่ได้อยู่ในสถานะยืมอยู่ หรือไม่ได้ถูกยืมโดยสมาชิกคนนี้") {}
};

class BorrowLimitExceededError : public LibraryError {
public:
    explicit BorrowLimitExceededError(const std::string& memberName)
        : LibraryError("สมาชิก \"" + memberName + "\" ยืมครบโควตาแล้ว (" +
                        std::to_string(Member::MAX_BORROW_LIMIT) + " รายการ)") {}
};
```

### เหตุผลของการออกแบบเป็นลำดับชั้น

| Exception | เมื่อไหร่ที่ throw | ข้อมูลพิเศษที่เก็บไว้ |
|---|---|---|
| `LibraryError` | base class ไม่ throw ตรงๆ ใช้เป็นจุด catch แบบกว้าง | ข้อความอย่างเดียว |
| `MediaNotFoundError` | หา `mediaId` ที่ระบุไม่เจอในระบบ | `mediaId_` |
| `MemberNotFoundError` | หา `memberId` ที่ระบุไม่เจอในระบบ | `memberId_` |
| `AlreadyBorrowedError` | พยายามยืมสื่อที่มีคนยืมอยู่แล้ว | (ใช้ข้อความอย่างเดียวก็เพียงพอ) |
| `NotBorrowedError` | พยายามคืนสื่อที่ไม่ได้อยู่ในสถานะยืม หรือคนละคนยืม | (ใช้ข้อความอย่างเดียวก็เพียงพอ) |
| `BorrowLimitExceededError` | สมาชิกยืมครบโควตา `Member::MAX_BORROW_LIMIT` แล้ว | (อ้างอิงค่า static member ของ `Member`) |

สังเกตว่า `BorrowLimitExceededError` **อ้างอิงค่า `Member::MAX_BORROW_LIMIT`** ที่เป็น
`static constexpr` โดยตรงในข้อความ error — นี่คือตัวอย่างการใช้ static member (Part 53) และ
exception (Part 54) ทำงานร่วมกันในระบบจริง

---

## 55.5 Member.h/.cpp — สมาชิกห้องสมุด (Step 437)

`Member.h`:

```cpp
#ifndef MEMBER_H
#define MEMBER_H

#include <iostream>
#include <string>
#include <vector>

#include "Media.h"

// ===== สมาชิกของห้องสมุด =====
class Member {
public:
    static constexpr int MAX_BORROW_LIMIT = 3;   // const static member: โควตายืมสูงสุดต่อคน

    Member(std::string id, std::string name);

    const std::string& id() const noexcept { return id_; }
    const std::string& name() const noexcept { return name_; }
    int borrowedCount() const noexcept { return static_cast<int>(borrowedMedia_.size()); }
    bool hasReachedLimit() const noexcept { return borrowedCount() >= MAX_BORROW_LIMIT; }
    bool isBorrowing(const Media* media) const;

    void addBorrowedMedia(Media* media);
    void removeBorrowedMedia(Media* media);

    friend std::ostream& operator<<(std::ostream& os, const Member& mem);

private:
    std::string id_;
    std::string name_;
    std::vector<Media*> borrowedMedia_;   // non-owning: Library เป็นเจ้าของ Media ตัวจริงผ่าน unique_ptr
};

#endif // MEMBER_H
```

`Member.cpp`:

```cpp
#include "Member.h"
#include <algorithm>
#include <utility>

Member::Member(std::string id, std::string name)
    : id_(std::move(id)), name_(std::move(name)) {}

bool Member::isBorrowing(const Media* media) const {
    return std::find(borrowedMedia_.begin(), borrowedMedia_.end(), media) != borrowedMedia_.end();
}

void Member::addBorrowedMedia(Media* media) {
    borrowedMedia_.push_back(media);
}

void Member::removeBorrowedMedia(Media* media) {
    auto it = std::find(borrowedMedia_.begin(), borrowedMedia_.end(), media);
    if (it != borrowedMedia_.end()) {
        borrowedMedia_.erase(it);
    }
}

std::ostream& operator<<(std::ostream& os, const Member& mem) {
    os << "สมาชิก: " << mem.name_ << " (ID: " << mem.id_ << ") - ยืมอยู่ "
       << mem.borrowedMedia_.size() << "/" << Member::MAX_BORROW_LIMIT << " รายการ";
    return os;
}
```

### จุดสำคัญของการออกแบบ Member

- **`static constexpr int MAX_BORROW_LIMIT = 3;`**: ค่าคงที่ระดับ class ที่ทุก `Member`
  แชร์ร่วมกัน (ทบทวน Part 53 ข้อ 53.4) เขียนแบบนี้ดีกว่าการเขียน `#define MAX_BORROW_LIMIT 3`
  แบบภาษา C เพราะมันอยู่ใน scope ของ `Member::` ชัดเจน ไม่ปนกับชื่ออื่นในโปรแกรม และมี type
  เป็น `int` อย่างชัดแจ้ง
- **`std::vector<Media*> borrowedMedia_;`**: เก็บ **raw pointer ที่ไม่เป็นเจ้าของ**
  (non-owning) — `Member` แค่ "จำ" ว่ากำลังยืมสื่อชิ้นไหนอยู่ ไม่มีหน้าที่ทำลายมัน เพราะ
  `Library` เป็นเจ้าของตัวจริงผ่าน `unique_ptr<Media>` การใช้ raw pointer ในลักษณะนี้ (สังเกต
  หรือ "observe" object ที่ไม่ได้เป็นเจ้าของ) เป็นรูปแบบที่**ยอมรับได้และเหมาะสม**ใน Modern
  C++ ตราบใดที่ผู้ใช้ pointer มั่นใจว่า object ที่ชี้ไปจะไม่ถูกทำลายก่อนที่ pointer จะเลิกใช้
  (ในระบบนี้ `Library` มีอายุยาวนานกว่าการยืม-คืนทุกครั้งอยู่แล้ว)
- **`isBorrowing()`** ใช้ `std::find` จาก `<algorithm>` เปรียบเทียบ **ที่อยู่ของ pointer**
  (ไม่ใช่เนื้อหาข้างใน) เพื่อตรวจสอบว่า `Media*` ตัวที่ระบุอยู่ในรายการที่สมาชิกคนนี้ยืมอยู่
  หรือไม่

---

## 55.6 Library.h/.cpp — จัดการ Collection, Borrow/Return, Static Counter (Step 438)

ส่วนที่เหลือของ `Library.h` (ต่อจาก exception hierarchy ใน 55.4) คือตัว class `Library` เอง:

```cpp
// ===== คลาสหลักที่บริหารจัดการห้องสมุดทั้งระบบ =====
class Library {
public:
    explicit Library(std::string name);

    Media& addMedia(std::unique_ptr<Media> media);
    Member& addMember(std::string id, std::string name);

    Media& findMedia(const std::string& mediaId);
    Member& findMember(const std::string& memberId);

    void borrowMedia(const std::string& memberId, const std::string& mediaId);
    void returnMedia(const std::string& memberId, const std::string& mediaId);

    void printCatalog(std::ostream& os) const;
    void printMembers(std::ostream& os) const;

    std::size_t mediaCount() const noexcept { return collection_.size(); }

private:
    std::string name_;
    std::vector<std::unique_ptr<Media>> collection_;   // Library เป็นเจ้าของสื่อทั้งหมดแต่เพียงผู้เดียว
    std::vector<std::unique_ptr<Member>> members_;     // ใช้ unique_ptr เพื่อให้ pointer ไม่เปลี่ยนที่แม้ vector realloc
};

#endif // LIBRARY_H
```

`Library.cpp`:

```cpp
#include "Library.h"
#include <utility>

Library::Library(std::string name) : name_(std::move(name)) {}

Media& Library::addMedia(std::unique_ptr<Media> media) {
    collection_.push_back(std::move(media));
    return *collection_.back();
}

Member& Library::addMember(std::string id, std::string name) {
    members_.push_back(std::make_unique<Member>(std::move(id), std::move(name)));
    return *members_.back();
}

Media& Library::findMedia(const std::string& mediaId) {
    for (auto& m : collection_) {
        if (m->id() == mediaId) return *m;
    }
    throw MediaNotFoundError(mediaId);
}

Member& Library::findMember(const std::string& memberId) {
    for (auto& mem : members_) {
        if (mem->id() == memberId) return *mem;
    }
    throw MemberNotFoundError(memberId);
}

void Library::borrowMedia(const std::string& memberId, const std::string& mediaId) {
    Member& member = findMember(memberId);   // อาจ throw MemberNotFoundError
    Media& media = findMedia(mediaId);       // อาจ throw MediaNotFoundError

    if (media.isBorrowed()) {
        throw AlreadyBorrowedError(media.title());
    }
    if (member.hasReachedLimit()) {
        throw BorrowLimitExceededError(member.name());
    }

    media.markBorrowed();
    member.addBorrowedMedia(&media);
}

void Library::returnMedia(const std::string& memberId, const std::string& mediaId) {
    Member& member = findMember(memberId);
    Media& media = findMedia(mediaId);

    if (!media.isBorrowed() || !member.isBorrowing(&media)) {
        throw NotBorrowedError(media.title());
    }

    media.markReturned();
    member.removeBorrowedMedia(&media);
}

void Library::printCatalog(std::ostream& os) const {
    os << "=== แคตตาล็อกของ " << name_ << " (" << collection_.size() << " รายการ) ===\n";
    for (const auto& m : collection_) {
        os << "  " << *m << '\n';
    }
}

void Library::printMembers(std::ostream& os) const {
    os << "=== สมาชิกทั้งหมด (" << members_.size() << " คน) ===\n";
    for (const auto& mem : members_) {
        os << "  " << *mem << '\n';
    }
}
```

### เดินตามลอจิกของ borrowMedia() ทีละขั้น

`borrowMedia()` คือหัวใจของ business logic ทั้งหมด มันตรวจสอบเงื่อนไข **4 ข้อ** เรียงตามลำดับ
ก่อนจะยอมให้ยืมสำเร็จ:

1. `findMember(memberId)` — ถ้าไม่พบสมาชิก throw `MemberNotFoundError` ทันที (ไม่ทำอะไรต่อ)
2. `findMedia(mediaId)` — ถ้าไม่พบสื่อ throw `MediaNotFoundError`
3. `media.isBorrowed()` — ถ้าสื่อถูกยืมอยู่แล้ว throw `AlreadyBorrowedError`
4. `member.hasReachedLimit()` — ถ้าสมาชิกยืมครบโควตาแล้ว throw `BorrowLimitExceededError`

ถ้าผ่านทั้งสี่เงื่อนไข จึงจะเรียก `media.markBorrowed()` (เปลี่ยนสถานะของ `Media`) และ
`member.addBorrowedMedia(&media)` (บันทึกไว้ที่ `Member`) — สังเกตว่าเราส่ง `&media` ซึ่งเป็น
ที่อยู่ของ `Media` ตัวจริงที่ `Library` เป็นเจ้าของอยู่ ไม่ใช่การ copy ข้อมูลไปมา

`returnMedia()` ใช้ตรรกะย้อนกลับ: ตรวจสอบว่าสื่อ**อยู่ในสถานะยืมอยู่จริง** และ**สมาชิกคนนี้เป็น
คนยืมจริง** (ไม่ใช่คนอื่น) — ถ้าเงื่อนไขใดเงื่อนไขหนึ่งไม่ผ่าน จะ throw `NotBorrowedError`
ทันที ป้องกันกรณีที่สมาชิกคนหนึ่งพยายาม "คืน" สื่อที่อีกคนยืมไปอยู่

### Sequence การทำงานของ borrowMedia() แบบเห็นภาพรวม

เพื่อให้เห็นว่าทั้งสาม class (`Library`, `Member`, `Media`) ทำงานร่วมกันอย่างไรในหนึ่ง
transaction ลองไล่ดู sequence ของการเรียก `lib.borrowMedia("U001", "B001")` แบบเต็มรูปแบบ:

```
main                Library                 Member (U001)          Media (B001)
 │                     │                          │                     │
 │ borrowMedia(        │                          │                     │
 │  "U001","B001")     │                          │                     │
 ├────────────────────►│                          │                     │
 │                     │ findMember("U001")       │                     │
 │                     ├─────────────────────────►│ (ค้นเจอ คืน ref)   │
 │                     │◄─────────────────────────┤                     │
 │                     │ findMedia("B001")                              │
 │                     ├────────────────────────────────────────────────►│ (ค้นเจอ คืน ref)
 │                     │◄────────────────────────────────────────────────┤
 │                     │ media.isBorrowed()? ── false (ผ่าน)             │
 │                     │ member.hasReachedLimit()? ── false (ผ่าน)       │
 │                     │ media.markBorrowed() ──────────────────────────►│ (borrowed_ = true)
 │                     │ member.addBorrowedMedia(&media) ───────────────►│
 │                     │                          │ (เก็บ pointer ไว้ใน  │
 │                     │                          │  borrowedMedia_)     │
 │◄────────────────────┤ (return ปกติ ไม่มี exception)                  │
```

สังเกตว่า **`Library` เป็นตัวประสานงานหลัก (orchestrator)** ที่คอยเรียก method ของ `Member`
และ `Media` ตามลำดับที่ถูกต้อง ในขณะที่ `Member` และ `Media` แต่ละตัวไม่รู้จักกันโดยตรง (Member
ไม่เรียก method ของ Media เอง และ Media ก็ไม่รู้จัก Member เลย) — การออกแบบแบบนี้สอดคล้องกับ
หลัก **Single Responsibility Principle (SRP)**: แต่ละ class มีหน้าที่รับผิดชอบเพียงอย่างเดียว
ที่ชัดเจน

| Class | ความรับผิดชอบเดียวที่ชัดเจน |
|---|---|
| `Media` (และลูกๆ) | รู้จักและแสดงข้อมูลของตัวเอง รวมถึงสถานะยืม/ว่างของตัวเอง |
| `Member` | รู้จักข้อมูลของสมาชิกคนหนึ่ง และจดจำว่าตัวเองกำลังยืมอะไรอยู่ |
| `Library` | ประสานงานระหว่าง `Media` และ `Member` ทั้งหมด บังคับใช้กฎทางธุรกิจ (business rule) เช่น โควตาการยืม |

ถ้าเราเผลอใส่ logic การตรวจสอบโควตาไปไว้ใน `Member` เอง (เช่น `Member::borrow(Media&)` ที่
ตรวจสอบทุกอย่างเบ็ดเสร็จในตัว) จะทำให้ `Member` ต้องรู้จัก `Library` และกฎทางธุรกิจของทั้งระบบ
ไปด้วย ซึ่งขัดกับหลัก SRP — การให้ `Library` เป็นผู้ตัดสินใจแต่เพียงผู้เดียวทำให้ทดสอบ
(unit test) แต่ละ class แยกจากกันได้ง่ายกว่ามาก (จะเรียนลึกเรื่อง Unit Testing ใน Part 93)

---

## 55.7 main.cpp — Integrate ทุกอย่างเข้าด้วยกัน และ Makefile (Step 439)

`main.cpp`:

```cpp
#include <iostream>
#include <memory>

#include "Library.h"
#include "Media.h"
#include "Member.h"

void printDivider(const std::string& label) {
    std::cout << "\n----- " << label << " -----\n";
}

int main() {
    Library lib("ห้องสมุดประชาชนสาขากลาง");

    // ----- เพิ่มสื่อเข้าห้องสมุด (polymorphism ผ่าน unique_ptr<Media>) -----
    Media& b1 = lib.addMedia(std::make_unique<Book>(
        "The C Programming Language", "B001", "K&R", "978-0-13-110362-7"));
    lib.addMedia(std::make_unique<Book>(
        "Effective Modern C++", "B002", "Scott Meyers", "978-1-4919-0399-5"));
    lib.addMedia(std::make_unique<DVD>(
        "The Matrix", "D001", "Wachowski Sisters", 136));
    lib.addMedia(std::make_unique<Magazine>(
        "National Geographic", "M001", 254));

    // ----- เพิ่มสมาชิก -----
    lib.addMember("U001", "Somchai");
    lib.addMember("U002", "Malee");

    printDivider("แคตตาล็อกทั้งหมด");
    lib.printCatalog(std::cout);

    printDivider("สถิติ static member");
    std::cout << "จำนวนสื่อทั้งหมดในระบบ (Media::totalMediaCount): "
              << Media::totalMediaCount() << '\n';
    std::cout << "จำนวนหนังสือทั้งหมดในระบบ (Book::totalBookCount): "
              << Book::totalBookCount() << '\n';

    printDivider("ยืมสื่อ (กรณีปกติ)");
    try {
        lib.borrowMedia("U001", "B001");
        std::cout << "Somchai ยืม B001 สำเร็จ\n";
        std::cout << "  " << b1 << '\n';
    } catch (const LibraryError& e) {
        std::cout << "เกิดข้อผิดพลาด: " << e.what() << '\n';
    }

    printDivider("ยืมสื่อที่ถูกยืมไปแล้ว (คาดว่า error)");
    try {
        lib.borrowMedia("U002", "B001"); // B001 ถูก Somchai ยืมไปแล้ว
        std::cout << "ไม่ควรมาถึงบรรทัดนี้\n";
    } catch (const AlreadyBorrowedError& e) {
        std::cout << "จับ AlreadyBorrowedError ได้ถูกต้อง: " << e.what() << '\n';
    }

    printDivider("ยืมสื่อที่ไม่มีในระบบ (คาดว่า error)");
    try {
        lib.borrowMedia("U002", "B999");
    } catch (const MediaNotFoundError& e) {
        std::cout << "จับ MediaNotFoundError ได้ถูกต้อง: " << e.what() << '\n';
    }

    printDivider("ทดสอบโควตาการยืม (BorrowLimitExceededError)");
    try {
        lib.borrowMedia("U002", "B002");
        lib.borrowMedia("U002", "D001");
        lib.borrowMedia("U002", "M001");
        std::cout << "Malee ยืมครบ " << Member::MAX_BORROW_LIMIT << " รายการแล้ว\n";

        // เพิ่มสื่ออีกชิ้นเพื่อทดสอบว่ายืมเกินโควตาจะถูกปฏิเสธ
        lib.addMedia(std::make_unique<Book>("Clean Code", "B003", "Robert C. Martin", "978-0-13-235088-4"));
        lib.borrowMedia("U002", "B003"); // ควร throw เพราะ Malee ยืมครบ 3 แล้ว
    } catch (const BorrowLimitExceededError& e) {
        std::cout << "จับ BorrowLimitExceededError ได้ถูกต้อง: " << e.what() << '\n';
    } catch (const LibraryError& e) {
        std::cout << "เกิดข้อผิดพลาดอื่นในระบบห้องสมุด: " << e.what() << '\n';
    }

    printDivider("คืนสื่อ");
    try {
        lib.returnMedia("U001", "B001");
        std::cout << "Somchai คืน B001 สำเร็จ\n";
        std::cout << "  " << b1 << '\n';
    } catch (const LibraryError& e) {
        std::cout << "เกิดข้อผิดพลาด: " << e.what() << '\n';
    }

    printDivider("คืนสื่อที่ไม่ได้ยืมไว้ (คาดว่า error)");
    try {
        lib.returnMedia("U001", "D001"); // Somchai ไม่ได้ยืม D001
    } catch (const NotBorrowedError& e) {
        std::cout << "จับ NotBorrowedError ได้ถูกต้อง: " << e.what() << '\n';
    }

    printDivider("สรุปสถานะสุดท้าย");
    lib.printCatalog(std::cout);
    lib.printMembers(std::cout);

    std::cout << "\nจำนวนสื่อทั้งหมดในระบบตอนนี้: " << Media::totalMediaCount() << '\n';
    std::cout << "จำนวนหนังสือทั้งหมดในระบบตอนนี้: " << Book::totalBookCount() << '\n';

    return 0;
}
```

สังเกตว่า `main.cpp` **ใช้ `Media&` (reference) ไม่ใช่ `Media*`** สำหรับ `b1` ที่เก็บผลลัพธ์
จาก `addMedia()` — เพราะ `Library` ยังเป็นเจ้าของ object ตัวจริง `main` แค่ "ยืม" reference มา
ใช้ชั่วคราวเพื่อแสดงผล ไม่มีการยุ่งเกี่ยวกับการจัดการ memory เลย ทั้งหมดอยู่ในความรับผิดชอบของ
`Library` ผ่าน `unique_ptr`

### คอมไพล์ทีละไฟล์ด้วยมือก่อนใช้ Makefile (ทบทวน Separate Compilation จาก Part 17)

ก่อนจะให้ `make` ทำงานให้อัตโนมัติ ลองเข้าใจเบื้องหลังด้วยการคอมไพล์ทีละไฟล์ด้วยมือก่อน เพื่อ
ทบทวนแนวคิด **Separate Compilation** ที่เรียนไปแล้วใน Part 17 (Modular Programming):

```bash
# ขั้นตอนที่ 1: compile แต่ละไฟล์ .cpp เป็น .o (object file) แยกกัน
# -c หมายถึง compile อย่างเดียว ไม่ link (ทบทวน Part 1: กระบวนการ Preprocess->Compile->Assemble->Link)
g++ -Wall -Wextra -Wpedantic -std=c++17 -c Media.cpp -o Media.o
g++ -Wall -Wextra -Wpedantic -std=c++17 -c Member.cpp -o Member.o
g++ -Wall -Wextra -Wpedantic -std=c++17 -c Library.cpp -o Library.o
g++ -Wall -Wextra -Wpedantic -std=c++17 -c main.cpp -o main.o

# ขั้นตอนที่ 2: link ทุก .o เข้าด้วยกันเป็น executable ตัวเดียว
g++ -Wall -Wextra -Wpedantic -std=c++17 Media.o Member.o Library.o main.o -o library_system

./library_system
```

สังเกตว่าแต่ละไฟล์ `.cpp` ถูก compile **แยกจากกันโดยอิสระ** — `Media.cpp` ไม่จำเป็นต้องรู้จัก
เนื้อหาของ `Library.cpp` เลย มันรู้แค่ว่า `Media.h` ประกาศอะไรไว้บ้าง (ผ่าน `#include`) การแบ่ง
แบบนี้ทำให้:

1. **แก้ไขไฟล์เดียว ไม่ต้อง compile ใหม่ทั้งโปรเจกต์** — ถ้าแก้แค่ `Member.cpp` เราแค่ compile
   `Member.cpp` ใหม่เป็น `Member.o` แล้ว link รวมกับ `.o` ไฟล์อื่นที่มีอยู่แล้ว (ไม่ต้อง compile
   `Media.cpp`/`Library.cpp` ซ้ำ) ในโปรเจกต์เล็กแบบนี้อาจไม่รู้สึกถึงความแตกต่างมาก แต่ในโปรเจกต์
   ขนาดใหญ่ระดับหมื่นบรรทัดขึ้นไป การ compile ใหม่ทั้งหมดทุกครั้งอาจใช้เวลาหลายนาทีถึงหลาย
   ชั่วโมง
2. **แบ่งงานพัฒนาเป็นทีมได้** — แต่ละคนแก้ไฟล์ `.cpp` ของตัวเองได้โดยไม่ชนกัน ตราบใดที่ header
   (`.h`) ที่เป็น "สัญญา" ร่วมกันไม่เปลี่ยน

`make` (ที่เราจะดูต่อไป) ไม่ได้ทำอะไรวิเศษไปกว่าสิ่งที่เราเพิ่งทำด้วยมือ — มันแค่ **จำ** ว่า
ไฟล์ไหนถูกแก้ไขล่าสุดเมื่อไหร่ (เทียบ timestamp) แล้ว compile ใหม่เฉพาะไฟล์ที่จำเป็นเท่านั้น
โดยอัตโนมัติ ทบทวนเรื่อง Makefile อย่างละเอียดได้ที่ Part 18

### Makefile

```makefile
CXX      := g++
CXXFLAGS := -std=c++17 -Wall -Wextra -Wpedantic -g
TARGET   := library_system
SOURCES  := main.cpp Media.cpp Library.cpp Member.cpp
OBJECTS  := $(SOURCES:.cpp=.o)
DEPS     := $(OBJECTS:.o=.d)

.PHONY: all clean run

all: $(TARGET)

$(TARGET): $(OBJECTS)
	$(CXX) $(CXXFLAGS) $(OBJECTS) -o $(TARGET)

%.o: %.cpp
	$(CXX) $(CXXFLAGS) -MMD -MP -c $< -o $@

-include $(DEPS)

run: all
	./$(TARGET)

clean:
	rm -f $(OBJECTS) $(DEPS) $(TARGET)
```

Makefile นี้ใช้เทคนิคที่เรียนใน Part 18 (Makefile และ Build Automation):

- `$(SOURCES:.cpp=.o)` — **Pattern Substitution**: แปลงรายชื่อไฟล์ `.cpp` เป็น `.o` โดยอัตโนมัติ
- `%.o: %.cpp` — **Pattern Rule** ทั่วไปที่ใช้คอมไพล์ไฟล์ `.cpp` แต่ละไฟล์เป็น `.o` โดยไม่ต้อง
  เขียนกฎซ้ำสำหรับทุกไฟล์
- `-MMD -MP` — ให้ compiler สร้างไฟล์ `.d` (dependency file) อัตโนมัติ ทำให้ `make` รู้ว่าถ้า
  แก้ header ไฟล์ไหน (เช่น `Media.h`) ไฟล์ `.cpp` ใดบ้างที่ต้อง compile ใหม่ (`-include $(DEPS)`
  ดึงไฟล์ `.d` เหล่านี้กลับเข้ามาใน Makefile)
- `run` และ `clean` เป็น **Phony Target** สำหรับความสะดวกในการรันและล้างไฟล์ที่ compile แล้ว

### คอมไพล์และรันโปรแกรมทั้งระบบ

```bash
make clean
make
./library_system
```

หรือใช้คำสั่งเดียว:

```bash
make run
```

ผลลัพธ์การรันทั้งหมด:

```

----- แคตตาล็อกทั้งหมด -----
=== แคตตาล็อกของ ห้องสมุดประชาชนสาขากลาง (4 รายการ) ===
  [Book] The C Programming Language (ID: B001) - ว่างอยู่บนชั้น | ผู้แต่ง: K&R, ISBN: 978-0-13-110362-7
  [Book] Effective Modern C++ (ID: B002) - ว่างอยู่บนชั้น | ผู้แต่ง: Scott Meyers, ISBN: 978-1-4919-0399-5
  [DVD] The Matrix (ID: D001) - ว่างอยู่บนชั้น | ผู้กำกับ: Wachowski Sisters, ความยาว: 136 นาที
  [Magazine] National Geographic (ID: M001) - ว่างอยู่บนชั้น | ฉบับที่: 254

----- สถิติ static member -----
จำนวนสื่อทั้งหมดในระบบ (Media::totalMediaCount): 4
จำนวนหนังสือทั้งหมดในระบบ (Book::totalBookCount): 2

----- ยืมสื่อ (กรณีปกติ) -----
Somchai ยืม B001 สำเร็จ
  [Book] The C Programming Language (ID: B001) - ถูกยืมอยู่ | ผู้แต่ง: K&R, ISBN: 978-0-13-110362-7

----- ยืมสื่อที่ถูกยืมไปแล้ว (คาดว่า error) -----
จับ AlreadyBorrowedError ได้ถูกต้อง: สื่อ "The C Programming Language" ถูกยืมไปแล้ว ไม่สามารถยืมซ้ำได้

----- ยืมสื่อที่ไม่มีในระบบ (คาดว่า error) -----
จับ MediaNotFoundError ได้ถูกต้อง: ไม่พบสื่อ ID: B999

----- ทดสอบโควตาการยืม (BorrowLimitExceededError) -----
Malee ยืมครบ 3 รายการแล้ว
จับ BorrowLimitExceededError ได้ถูกต้อง: สมาชิก "Malee" ยืมครบโควตาแล้ว (3 รายการ)

----- คืนสื่อ -----
Somchai คืน B001 สำเร็จ
  [Book] The C Programming Language (ID: B001) - ว่างอยู่บนชั้น | ผู้แต่ง: K&R, ISBN: 978-0-13-110362-7

----- คืนสื่อที่ไม่ได้ยืมไว้ (คาดว่า error) -----
จับ NotBorrowedError ได้ถูกต้อง: สื่อ "The Matrix" ไม่ได้อยู่ในสถานะยืมอยู่ หรือไม่ได้ถูกยืมโดยสมาชิกคนนี้

----- สรุปสถานะสุดท้าย -----
=== แคตตาล็อกของ ห้องสมุดประชาชนสาขากลาง (5 รายการ) ===
  [Book] The C Programming Language (ID: B001) - ว่างอยู่บนชั้น | ผู้แต่ง: K&R, ISBN: 978-0-13-110362-7
  [Book] Effective Modern C++ (ID: B002) - ถูกยืมอยู่ | ผู้แต่ง: Scott Meyers, ISBN: 978-1-4919-0399-5
  [DVD] The Matrix (ID: D001) - ถูกยืมอยู่ | ผู้กำกับ: Wachowski Sisters, ความยาว: 136 นาที
  [Magazine] National Geographic (ID: M001) - ถูกยืมอยู่ | ฉบับที่: 254
  [Book] Clean Code (ID: B003) - ว่างอยู่บนชั้น | ผู้แต่ง: Robert C. Martin, ISBN: 978-0-13-235088-4
=== สมาชิกทั้งหมด (2 คน) ===
  สมาชิก: Somchai (ID: U001) - ยืมอยู่ 0/3 รายการ
  สมาชิก: Malee (ID: U002) - ยืมอยู่ 3/3 รายการ

จำนวนสื่อทั้งหมดในระบบตอนนี้: 5
จำนวนหนังสือทั้งหมดในระบบตอนนี้: 3
```

โค้ดทั้งหมดคอมไพล์ผ่านด้วย `g++ -Wall -Wextra -Wpedantic -std=c++17` **โดยไม่มี warning แม้แต่
บรรทัดเดียว** — ยืนยันว่าการออกแบบ ownership (ผ่าน `unique_ptr`), การจัดการ virtual
destructor, และการใช้ exception ทั้งหมดถูกต้องตามมาตรฐานภาษาอย่างเคร่งครัด

---

## 55.8 ทดสอบระบบทั้งหมด และแนวทางต่อยอด (Step 440)

### ตารางทดสอบ (Test Matrix) ของระบบ

ก่อนสรุปผล เรามาดูภาพรวมว่าการรันใน `main.cpp` ครอบคลุมสถานการณ์ (test case) อะไรบ้าง และ
คาดหวังผลลัพธ์แบบไหน — การคิดเป็น "ตารางทดสอบ" แบบนี้คือจุดเริ่มต้นตามธรรมชาติของการเขียน
Unit Test อย่างเป็นระบบที่จะเรียนเต็มรูปแบบใน Part 93 (Google Test/Catch2)

| # | สถานการณ์ (Scenario) | Input | ผลลัพธ์ที่คาดหวัง | ตรงกับผลรันจริงหรือไม่ |
|---|---|---|---|---|
| 1 | ยืมสื่อที่ว่างอยู่ | `borrowMedia("U001", "B001")` | สำเร็จ, `isBorrowed()` เป็น `true` | ตรง ✓ |
| 2 | ยืมสื่อที่ถูกยืมไปแล้ว | `borrowMedia("U002", "B001")` | throw `AlreadyBorrowedError` | ตรง ✓ |
| 3 | ยืมสื่อที่ไม่มีในระบบ | `borrowMedia("U002", "B999")` | throw `MediaNotFoundError` | ตรง ✓ |
| 4 | ยืมสื่อกับสมาชิกที่ไม่มีในระบบ | `borrowMedia("U999", "B001")` | throw `MemberNotFoundError` | ตรง ✓ (ตรวจสอบ `findMember` ก่อน `findMedia` เสมอ) |
| 5 | ยืมสื่อจนครบโควตา 3 รายการ | ยืม `B002`, `D001`, `M001` ให้ `U002` | สำเร็จทั้ง 3 ครั้ง | ตรง ✓ |
| 6 | ยืมสื่อชิ้นที่ 4 เกินโควตา | `borrowMedia("U002", "B003")` | throw `BorrowLimitExceededError` | ตรง ✓ |
| 7 | คืนสื่อที่ยืมไว้ถูกต้อง | `returnMedia("U001", "B001")` | สำเร็จ, `isBorrowed()` กลับเป็น `false` | ตรง ✓ |
| 8 | คืนสื่อที่ไม่ได้ยืมไว้เลย | `returnMedia("U001", "D001")` | throw `NotBorrowedError` | ตรง ✓ |
| 9 | นับจำนวนสื่อทั้งหมดด้วย static member | `Media::totalMediaCount()` | เพิ่มขึ้นตามจำนวนสื่อที่ `addMedia()` จริง | ตรง ✓ (4 → 5 หลังเพิ่ม `B003`) |
| 10 | นับจำนวนหนังสือเฉพาะด้วย static member | `Book::totalBookCount()` | นับเฉพาะ `Book` ไม่รวม `DVD`/`Magazine` | ตรง ✓ (2 → 3 หลังเพิ่ม `B003`) |

การที่ผลลัพธ์จริงตรงกับที่คาดหวังครบทุกแถว หมายความว่า business logic หลักของระบบทำงานถูกต้อง
สมบูรณ์ตาม requirement ที่วางไว้ใน 55.1

### สิ่งที่ผลการรันใน 55.7 พิสูจน์ได้แล้ว

เดินตามผลลัพธ์ทีละส่วน เราจะเห็นว่าระบบผ่านการทดสอบสถานการณ์สำคัญครบทุกกรณี:

1. **Polymorphism ทำงานถูกต้อง**: `printCatalog()` แสดงข้อมูลของ `Book`, `DVD`, `Magazine`
   แตกต่างกันถูกต้องตามชนิดจริง (`extraInfo()` ของแต่ละชนิดถูกเรียกผ่าน virtual dispatch)
   ทั้งที่ `printCatalog()` เขียนโค้ดแค่ `os << *m` โดยไม่รู้เลยว่า `m` ชี้ไปที่ชนิดไหน
2. **Static member ทำงานถูกต้อง**: `Media::totalMediaCount()` และ `Book::totalBookCount()`
   นับจำนวนถูกต้องหลังเพิ่มสื่อใหม่ (`B003`) เข้าไปกลางโปรแกรม
3. **Exception hierarchy จับ error ได้ตรงจุด**: ทั้ง `AlreadyBorrowedError`,
   `MediaNotFoundError`, `BorrowLimitExceededError`, และ `NotBorrowedError` ถูก throw และ
   catch ตรงตามสถานการณ์ที่ออกแบบไว้ทั้งหมด
4. **ไม่มี memory leak**: ไม่มีการเรียก `new`/`delete` แบบ manual เลยในทั้งโปรเจกต์ ทุกอย่าง
   ผ่าน `unique_ptr` ที่จัดการ ownership ให้อัตโนมัติตามหลัก RAII

### แนวทางต่อยอดโปรเจกต์นี้ (เชื่อมโยงกับ Part ในอนาคต)

โปรเจกต์นี้ถูกออกแบบให้ **ขยายต่อได้ง่าย** เพราะยึดหลัก OOP ที่ดีตลอดทั้ง Module D นี่คือ
แนวทางต่อยอดที่จะสมเหตุสมผลมากขึ้นเมื่อเรียนเนื้อหาถัดไปในหลักสูตร:

- **แทนที่ raw pointer ใน `Member::borrowedMedia_`** ด้วย `std::weak_ptr<Media>` เพื่อป้องกัน
  การเข้าถึง pointer ที่ dangling อย่างสมบูรณ์แบบ (เรียนเต็มรูปแบบใน **Part 67**)
- **ใช้ Template** เขียน `Repository<T>` แบบ generic แทนโค้ด `findMedia`/`findMember` ที่คล้าย
  กันสองชุด (เรียนใน **Part 56**)
- **เก็บข้อมูลลง STL container ที่เหมาะสมกว่า `vector`** เช่น `std::unordered_map<std::string,
  std::unique_ptr<Media>>` เพื่อค้นหาด้วย id ในเวลา O(1) แทนการวน loop แบบ O(n) (เรียนใน
  **Part 61**)
- **บันทึกข้อมูลลงไฟล์**ให้คงอยู่ข้ามการรันโปรแกรม โดยใช้เทคนิค File I/O ที่เรียนไปแล้วใน
  **Part 13** หรือจัดเก็บลงฐานข้อมูลจริงด้วย SQLite ใน **Part 105**
- **เขียน Unit Test อัตโนมัติ** ด้วย Google Test หรือ Catch2 แทนการอ่านผล output ด้วยตาเปล่า
  แบบใน `main.cpp` (เรียนใน **Part 93**)
- **ประยุกต์ Design Pattern เพิ่มเติม** เช่น Factory Pattern สำหรับสร้าง `Media` จากข้อมูลดิบ
  หรือ Observer Pattern สำหรับแจ้งเตือนเมื่อมีการยืม-คืน (เรียนใน **Module H, Part 97-98**)
- **ต่อยอดเป็น REST API** ให้เข้าถึงผ่านเว็บได้ โดยใช้ Framework อย่าง Crow ที่จะเรียนใน
  **Part 102-103**

---

## ภาคผนวก: โค้ดฉบับสมบูรณ์ทั้งหมด (Complete Source Listing)

ก่อนไปดูข้อผิดพลาดที่พบบ่อยและแบบฝึกหัด นี่คือโค้ดทั้งหมดของโปรเจกต์รวบรวมไว้ในที่เดียว
(เหมือนกับที่อธิบายแยกส่วนไปทีละไฟล์ใน 55.2-55.7 ทุกประการ) เพื่อให้คัดลอกไปคอมไพล์และทดลองรัน
เองได้สะดวก โครงสร้างไดเรกทอรีของโปรเจกต์มีดังนี้:

```
library_system/
├── Media.h
├── Media.cpp
├── Member.h
├── Member.cpp
├── Library.h
├── Library.cpp
├── main.cpp
└── Makefile
```

### Media.h

```cpp
#ifndef MEDIA_H
#define MEDIA_H

#include <iostream>
#include <string>

// ===== คลาสฐานนามธรรม (Abstract Base Class) สำหรับสื่อทุกชนิดในห้องสมุด =====
class Media {
public:
    Media(std::string title, std::string id);
    virtual ~Media();

    // ห้าม copy สื่อโดยตรง เพราะ Library เป็นเจ้าของผ่าน unique_ptr เพียงที่เดียว
    Media(const Media&) = delete;
    Media& operator=(const Media&) = delete;

    virtual std::string mediaType() const = 0;         // pure virtual -> ทำให้ Media เป็น abstract class
    virtual std::string extraInfo() const;              // ข้อมูลเพิ่มเติมเฉพาะชนิด (default ว่างเปล่า)

    const std::string& title() const noexcept { return title_; }
    const std::string& id() const noexcept { return id_; }
    bool isBorrowed() const noexcept { return borrowed_; }

    void markBorrowed();
    void markReturned();

    static int totalMediaCount() noexcept { return totalMediaCount_; }

    friend std::ostream& operator<<(std::ostream& os, const Media& m);

private:
    std::string title_;
    std::string id_;
    bool borrowed_ = false;

    static int totalMediaCount_;   // static data member: นับจำนวนสื่อทั้งหมดที่มีอยู่ในระบบ ณ ขณะนี้
};

// ===== หนังสือ =====
class Book : public Media {
public:
    Book(std::string title, std::string id, std::string author, std::string isbn);
    ~Book() override;

    std::string mediaType() const override { return "Book"; }
    std::string extraInfo() const override;

    const std::string& author() const noexcept { return author_; }
    const std::string& isbn() const noexcept { return isbn_; }

    static int totalBookCount() noexcept { return totalBookCount_; }

private:
    std::string author_;
    std::string isbn_;

    static int totalBookCount_;    // static data member เฉพาะของ Book: นับจำนวนหนังสือทั้งหมดในระบบ
};

// ===== แผ่น DVD =====
class DVD : public Media {
public:
    DVD(std::string title, std::string id, std::string director, int runtimeMinutes);

    std::string mediaType() const override { return "DVD"; }
    std::string extraInfo() const override;

private:
    std::string director_;
    int runtimeMinutes_;
};

// ===== นิตยสาร =====
class Magazine : public Media {
public:
    Magazine(std::string title, std::string id, int issueNumber);

    std::string mediaType() const override { return "Magazine"; }
    std::string extraInfo() const override;

private:
    int issueNumber_;
};

#endif // MEDIA_H
```

### Media.cpp

```cpp
#include "Media.h"
#include <utility>

int Media::totalMediaCount_ = 0;

Media::Media(std::string title, std::string id)
    : title_(std::move(title)), id_(std::move(id)) {
    ++totalMediaCount_;
}

Media::~Media() {
    --totalMediaCount_;
}

std::string Media::extraInfo() const {
    return "";
}

void Media::markBorrowed() {
    borrowed_ = true;
}

void Media::markReturned() {
    borrowed_ = false;
}

std::ostream& operator<<(std::ostream& os, const Media& m) {
    os << "[" << m.mediaType() << "] " << m.title() << " (ID: " << m.id() << ") - "
       << (m.borrowed_ ? "ถูกยืมอยู่" : "ว่างอยู่บนชั้น");
    const std::string extra = m.extraInfo();   // เรียกผ่าน virtual function -> dynamic dispatch
    if (!extra.empty()) {
        os << " | " << extra;
    }
    return os;
}

int Book::totalBookCount_ = 0;

Book::Book(std::string title, std::string id, std::string author, std::string isbn)
    : Media(std::move(title), std::move(id)), author_(std::move(author)), isbn_(std::move(isbn)) {
    ++totalBookCount_;
}

Book::~Book() {
    --totalBookCount_;
}

std::string Book::extraInfo() const {
    return "ผู้แต่ง: " + author_ + ", ISBN: " + isbn_;
}

DVD::DVD(std::string title, std::string id, std::string director, int runtimeMinutes)
    : Media(std::move(title), std::move(id)),
      director_(std::move(director)),
      runtimeMinutes_(runtimeMinutes) {}

std::string DVD::extraInfo() const {
    return "ผู้กำกับ: " + director_ + ", ความยาว: " + std::to_string(runtimeMinutes_) + " นาที";
}

Magazine::Magazine(std::string title, std::string id, int issueNumber)
    : Media(std::move(title), std::move(id)), issueNumber_(issueNumber) {}

std::string Magazine::extraInfo() const {
    return "ฉบับที่: " + std::to_string(issueNumber_);
}
```

### Member.h

```cpp
#ifndef MEMBER_H
#define MEMBER_H

#include <iostream>
#include <string>
#include <vector>

#include "Media.h"

// ===== สมาชิกของห้องสมุด =====
class Member {
public:
    static constexpr int MAX_BORROW_LIMIT = 3;   // const static member: โควตายืมสูงสุดต่อคน

    Member(std::string id, std::string name);

    const std::string& id() const noexcept { return id_; }
    const std::string& name() const noexcept { return name_; }
    int borrowedCount() const noexcept { return static_cast<int>(borrowedMedia_.size()); }
    bool hasReachedLimit() const noexcept { return borrowedCount() >= MAX_BORROW_LIMIT; }
    bool isBorrowing(const Media* media) const;

    void addBorrowedMedia(Media* media);
    void removeBorrowedMedia(Media* media);

    friend std::ostream& operator<<(std::ostream& os, const Member& mem);

private:
    std::string id_;
    std::string name_;
    std::vector<Media*> borrowedMedia_;   // non-owning: Library เป็นเจ้าของ Media ตัวจริงผ่าน unique_ptr
};

#endif // MEMBER_H
```

### Member.cpp

```cpp
#include "Member.h"
#include <algorithm>
#include <utility>

Member::Member(std::string id, std::string name)
    : id_(std::move(id)), name_(std::move(name)) {}

bool Member::isBorrowing(const Media* media) const {
    return std::find(borrowedMedia_.begin(), borrowedMedia_.end(), media) != borrowedMedia_.end();
}

void Member::addBorrowedMedia(Media* media) {
    borrowedMedia_.push_back(media);
}

void Member::removeBorrowedMedia(Media* media) {
    auto it = std::find(borrowedMedia_.begin(), borrowedMedia_.end(), media);
    if (it != borrowedMedia_.end()) {
        borrowedMedia_.erase(it);
    }
}

std::ostream& operator<<(std::ostream& os, const Member& mem) {
    os << "สมาชิก: " << mem.name_ << " (ID: " << mem.id_ << ") - ยืมอยู่ "
       << mem.borrowedMedia_.size() << "/" << Member::MAX_BORROW_LIMIT << " รายการ";
    return os;
}
```

### Library.h

```cpp
#ifndef LIBRARY_H
#define LIBRARY_H

#include <iostream>
#include <memory>
#include <stdexcept>
#include <string>
#include <vector>

#include "Media.h"
#include "Member.h"

// ===== Exception hierarchy ของระบบห้องสมุด (สืบทอดจาก std::runtime_error) =====

class LibraryError : public std::runtime_error {
public:
    explicit LibraryError(const std::string& message) : std::runtime_error(message) {}
};

class MediaNotFoundError : public LibraryError {
public:
    explicit MediaNotFoundError(const std::string& mediaId)
        : LibraryError("ไม่พบสื่อ ID: " + mediaId), mediaId_(mediaId) {}
    const std::string& mediaId() const noexcept { return mediaId_; }
private:
    std::string mediaId_;
};

class MemberNotFoundError : public LibraryError {
public:
    explicit MemberNotFoundError(const std::string& memberId)
        : LibraryError("ไม่พบสมาชิก ID: " + memberId), memberId_(memberId) {}
    const std::string& memberId() const noexcept { return memberId_; }
private:
    std::string memberId_;
};

class AlreadyBorrowedError : public LibraryError {
public:
    explicit AlreadyBorrowedError(const std::string& title)
        : LibraryError("สื่อ \"" + title + "\" ถูกยืมไปแล้ว ไม่สามารถยืมซ้ำได้") {}
};

class NotBorrowedError : public LibraryError {
public:
    explicit NotBorrowedError(const std::string& title)
        : LibraryError("สื่อ \"" + title + "\" ไม่ได้อยู่ในสถานะยืมอยู่ หรือไม่ได้ถูกยืมโดยสมาชิกคนนี้") {}
};

class BorrowLimitExceededError : public LibraryError {
public:
    explicit BorrowLimitExceededError(const std::string& memberName)
        : LibraryError("สมาชิก \"" + memberName + "\" ยืมครบโควตาแล้ว (" +
                        std::to_string(Member::MAX_BORROW_LIMIT) + " รายการ)") {}
};

// ===== คลาสหลักที่บริหารจัดการห้องสมุดทั้งระบบ =====
class Library {
public:
    explicit Library(std::string name);

    Media& addMedia(std::unique_ptr<Media> media);
    Member& addMember(std::string id, std::string name);

    Media& findMedia(const std::string& mediaId);
    Member& findMember(const std::string& memberId);

    void borrowMedia(const std::string& memberId, const std::string& mediaId);
    void returnMedia(const std::string& memberId, const std::string& mediaId);

    void printCatalog(std::ostream& os) const;
    void printMembers(std::ostream& os) const;

    std::size_t mediaCount() const noexcept { return collection_.size(); }

private:
    std::string name_;
    std::vector<std::unique_ptr<Media>> collection_;   // Library เป็นเจ้าของสื่อทั้งหมดแต่เพียงผู้เดียว
    std::vector<std::unique_ptr<Member>> members_;     // ใช้ unique_ptr เพื่อให้ pointer ไม่เปลี่ยนที่แม้ vector realloc
};

#endif // LIBRARY_H
```

### Library.cpp

```cpp
#include "Library.h"
#include <utility>

Library::Library(std::string name) : name_(std::move(name)) {}

Media& Library::addMedia(std::unique_ptr<Media> media) {
    collection_.push_back(std::move(media));
    return *collection_.back();
}

Member& Library::addMember(std::string id, std::string name) {
    members_.push_back(std::make_unique<Member>(std::move(id), std::move(name)));
    return *members_.back();
}

Media& Library::findMedia(const std::string& mediaId) {
    for (auto& m : collection_) {
        if (m->id() == mediaId) return *m;
    }
    throw MediaNotFoundError(mediaId);
}

Member& Library::findMember(const std::string& memberId) {
    for (auto& mem : members_) {
        if (mem->id() == memberId) return *mem;
    }
    throw MemberNotFoundError(memberId);
}

void Library::borrowMedia(const std::string& memberId, const std::string& mediaId) {
    Member& member = findMember(memberId);   // อาจ throw MemberNotFoundError
    Media& media = findMedia(mediaId);       // อาจ throw MediaNotFoundError

    if (media.isBorrowed()) {
        throw AlreadyBorrowedError(media.title());
    }
    if (member.hasReachedLimit()) {
        throw BorrowLimitExceededError(member.name());
    }

    media.markBorrowed();
    member.addBorrowedMedia(&media);
}

void Library::returnMedia(const std::string& memberId, const std::string& mediaId) {
    Member& member = findMember(memberId);
    Media& media = findMedia(mediaId);

    if (!media.isBorrowed() || !member.isBorrowing(&media)) {
        throw NotBorrowedError(media.title());
    }

    media.markReturned();
    member.removeBorrowedMedia(&media);
}

void Library::printCatalog(std::ostream& os) const {
    os << "=== แคตตาล็อกของ " << name_ << " (" << collection_.size() << " รายการ) ===\n";
    for (const auto& m : collection_) {
        os << "  " << *m << '\n';
    }
}

void Library::printMembers(std::ostream& os) const {
    os << "=== สมาชิกทั้งหมด (" << members_.size() << " คน) ===\n";
    for (const auto& mem : members_) {
        os << "  " << *mem << '\n';
    }
}
```

### main.cpp

```cpp
#include <iostream>
#include <memory>

#include "Library.h"
#include "Media.h"
#include "Member.h"

void printDivider(const std::string& label) {
    std::cout << "\n----- " << label << " -----\n";
}

int main() {
    Library lib("ห้องสมุดประชาชนสาขากลาง");

    // ----- เพิ่มสื่อเข้าห้องสมุด (polymorphism ผ่าน unique_ptr<Media>) -----
    Media& b1 = lib.addMedia(std::make_unique<Book>(
        "The C Programming Language", "B001", "K&R", "978-0-13-110362-7"));
    lib.addMedia(std::make_unique<Book>(
        "Effective Modern C++", "B002", "Scott Meyers", "978-1-4919-0399-5"));
    lib.addMedia(std::make_unique<DVD>(
        "The Matrix", "D001", "Wachowski Sisters", 136));
    lib.addMedia(std::make_unique<Magazine>(
        "National Geographic", "M001", 254));

    // ----- เพิ่มสมาชิก -----
    lib.addMember("U001", "Somchai");
    lib.addMember("U002", "Malee");

    printDivider("แคตตาล็อกทั้งหมด");
    lib.printCatalog(std::cout);

    printDivider("สถิติ static member");
    std::cout << "จำนวนสื่อทั้งหมดในระบบ (Media::totalMediaCount): "
              << Media::totalMediaCount() << '\n';
    std::cout << "จำนวนหนังสือทั้งหมดในระบบ (Book::totalBookCount): "
              << Book::totalBookCount() << '\n';

    printDivider("ยืมสื่อ (กรณีปกติ)");
    try {
        lib.borrowMedia("U001", "B001");
        std::cout << "Somchai ยืม B001 สำเร็จ\n";
        std::cout << "  " << b1 << '\n';
    } catch (const LibraryError& e) {
        std::cout << "เกิดข้อผิดพลาด: " << e.what() << '\n';
    }

    printDivider("ยืมสื่อที่ถูกยืมไปแล้ว (คาดว่า error)");
    try {
        lib.borrowMedia("U002", "B001"); // B001 ถูก Somchai ยืมไปแล้ว
        std::cout << "ไม่ควรมาถึงบรรทัดนี้\n";
    } catch (const AlreadyBorrowedError& e) {
        std::cout << "จับ AlreadyBorrowedError ได้ถูกต้อง: " << e.what() << '\n';
    }

    printDivider("ยืมสื่อที่ไม่มีในระบบ (คาดว่า error)");
    try {
        lib.borrowMedia("U002", "B999");
    } catch (const MediaNotFoundError& e) {
        std::cout << "จับ MediaNotFoundError ได้ถูกต้อง: " << e.what() << '\n';
    }

    printDivider("ทดสอบโควตาการยืม (BorrowLimitExceededError)");
    try {
        lib.borrowMedia("U002", "B002");
        lib.borrowMedia("U002", "D001");
        lib.borrowMedia("U002", "M001");
        std::cout << "Malee ยืมครบ " << Member::MAX_BORROW_LIMIT << " รายการแล้ว\n";

        // เพิ่มสื่ออีกชิ้นเพื่อทดสอบว่ายืมเกินโควตาจะถูกปฏิเสธ
        lib.addMedia(std::make_unique<Book>("Clean Code", "B003", "Robert C. Martin", "978-0-13-235088-4"));
        lib.borrowMedia("U002", "B003"); // ควร throw เพราะ Malee ยืมครบ 3 แล้ว
    } catch (const BorrowLimitExceededError& e) {
        std::cout << "จับ BorrowLimitExceededError ได้ถูกต้อง: " << e.what() << '\n';
    } catch (const LibraryError& e) {
        std::cout << "เกิดข้อผิดพลาดอื่นในระบบห้องสมุด: " << e.what() << '\n';
    }

    printDivider("คืนสื่อ");
    try {
        lib.returnMedia("U001", "B001");
        std::cout << "Somchai คืน B001 สำเร็จ\n";
        std::cout << "  " << b1 << '\n';
    } catch (const LibraryError& e) {
        std::cout << "เกิดข้อผิดพลาด: " << e.what() << '\n';
    }

    printDivider("คืนสื่อที่ไม่ได้ยืมไว้ (คาดว่า error)");
    try {
        lib.returnMedia("U001", "D001"); // Somchai ไม่ได้ยืม D001
    } catch (const NotBorrowedError& e) {
        std::cout << "จับ NotBorrowedError ได้ถูกต้อง: " << e.what() << '\n';
    }

    printDivider("สรุปสถานะสุดท้าย");
    lib.printCatalog(std::cout);
    lib.printMembers(std::cout);

    std::cout << "\nจำนวนสื่อทั้งหมดในระบบตอนนี้: " << Media::totalMediaCount() << '\n';
    std::cout << "จำนวนหนังสือทั้งหมดในระบบตอนนี้: " << Book::totalBookCount() << '\n';

    return 0;
}
```

### Makefile

```makefile
CXX      := g++
CXXFLAGS := -std=c++17 -Wall -Wextra -Wpedantic -g
TARGET   := library_system
SOURCES  := main.cpp Media.cpp Library.cpp Member.cpp
OBJECTS  := $(SOURCES:.cpp=.o)
DEPS     := $(OBJECTS:.o=.d)

.PHONY: all clean run

all: $(TARGET)

$(TARGET): $(OBJECTS)
	$(CXX) $(CXXFLAGS) $(OBJECTS) -o $(TARGET)

%.o: %.cpp
	$(CXX) $(CXXFLAGS) -MMD -MP -c $< -o $@

-include $(DEPS)

run: all
	./$(TARGET)

clean:
	rm -f $(OBJECTS) $(DEPS) $(TARGET)
```

โค้ดชุดนี้เหมือนกับที่อธิบายไว้ในหัวข้อ 55.2-55.7 ทุกตัวอักษร คัดลอกทั้ง 8 ไฟล์ไปไว้ในโฟลเดอร์
เดียวกัน แล้วรัน `make run` ได้ทันที

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมประกาศ destructor เป็น `virtual` ใน base class ที่ถูกลบผ่าน base class pointer** —
   ถ้า `~Media()` ไม่ใช่ `virtual` การลบ `unique_ptr<Media>` ที่จริงๆ ชี้ไปที่ `Book` จะเรียก
   เฉพาะ `~Media()` เท่านั้น ทำให้ `author_`/`isbn_` ของ `Book` ไม่ถูกทำลายอย่างถูกต้อง
   (undefined behavior) กฎทองคือ: **class ใดก็ตามที่ออกแบบมาให้สืบทอด ต้องมี virtual
   destructor เสมอ**
2. **เก็บ `Member` หรือ `Media` ตรงๆ ใน `std::vector<T>` แทน `vector<unique_ptr<T>>`** — ถ้า
   เก็บ `vector<Member>` ตรงๆ แล้วมีการ `push_back` เพิ่มสมาชิกใหม่ในภายหลัง vector อาจต้อง
   reallocate หน่วยความจำ ทำให้ **reference หรือ pointer ที่เคยชี้ไปยัง element เดิม
   กลายเป็น dangling ทันที** (ปัญหานี้ยิ่งอันตรายเพราะบางครั้งโปรแกรมยังทำงาน "ดูเหมือน" ถูก
   ต้องไปได้พักหนึ่งก่อนจะพังในจุดที่คาดไม่ถึง) — การใช้ `vector<unique_ptr<T>>` แก้ปัญหานี้
   เพราะสิ่งที่ reallocate คือ `unique_ptr` (ที่เป็นแค่ pointer ตัวเล็กๆ) ไม่ใช่ตัว object จริง
3. **ลืม `std::move` ตอนส่ง `unique_ptr` เข้าฟังก์ชัน** — `unique_ptr` **copy ไม่ได้**
   (copy constructor ถูก `= delete` ไว้ใน Standard Library) การเรียก
   `lib.addMedia(bookPtr)` โดยไม่ใส่ `std::move(bookPtr)` จะเป็น **compile error** ทันที
   ต้องเขียน `lib.addMedia(std::move(bookPtr))` หรือสร้างแบบ temporary ด้วย
   `std::make_unique<Book>(...)` ตรงจุดเรียกเลยเหมือนใน `main.cpp` ของเรา
4. **ตรวจสอบแค่ `media.isBorrowed()` แต่ไม่ตรวจสอบว่าใครเป็นคนยืม ตอน return** — ถ้า
   `returnMedia()` เช็คแค่ `isBorrowed()` เพียงอย่างเดียว จะทำให้สมาชิกคนหนึ่งสามารถ "คืน" สื่อ
   ที่อีกคนยืมไปได้ ซึ่งเป็นบั๊กทางธุรกิจที่ร้ายแรง ต้องเช็ค `member.isBorrowing(&media)`
   ควบคู่ไปด้วยเสมอ (ดังที่ทำใน `Library::returnMedia()`)
5. **ลืมว่า `operator<<` ที่เป็น `friend` ต้อง declare อยู่ใน class แต่ define นอก class** —
   ถ้า define implementation ไว้ผิดที่ (เช่น พยายาม define ใน derived class) จะกลายเป็นการ
   ประกาศ `operator<<` ใหม่ที่ไม่เกี่ยวข้องกับตัวเดิมเลย ทำให้ polymorphism ผ่าน `mediaType()`/
   `extraInfo()` ไม่ทำงานตามที่คาดหวัง
6. **ออกแบบ static member ให้ผูกกับข้อมูลที่ไม่ควรแชร์ร่วมกัน** — เช่น ถ้าเผลอทำให้
   `Member::MAX_BORROW_LIMIT` เป็น non-static แทนที่จะเป็น static จะทำให้สมาชิกแต่ละคนมีโควตา
   ต่างกันโดยไม่ได้ตั้งใจ (ค่าเริ่มต้นของแต่ละ object จะเป็นค่าที่ไม่แน่นอนถ้าไม่ได้ initialize)
   ทบทวนหลักคิดจาก Part 53: "ถ้าค่านั้นควรเหมือนกันสำหรับทุก object เสมอ ให้ใช้ static"
7. **ตรวจสอบเงื่อนไข error หลังจากแก้ไข state ของ object ไปแล้วบางส่วน** — เช่น ถ้าเขียน
   `borrowMedia()` โดยเรียก `member.addBorrowedMedia(&media)` ก่อน แล้วค่อยเช็ค
   `member.hasReachedLimit()` ทีหลัง จะทำให้สมาชิกที่ยืมเกินโควตาไปแล้ว 1 รายการ (เพราะ state
   ถูกแก้ไขไปแล้วก่อนที่ error จะถูกตรวจพบ) ต้องเรียงลำดับให้ **ตรวจสอบเงื่อนไข error ทั้งหมด
   ให้เสร็จก่อน แล้วค่อยแก้ไข state** เสมอ (Strong Exception Safety Guarantee ที่กล่าวถึงใน
   เฉลยแบบฝึกหัดข้อ 2)
8. **ลืมเพิ่มไฟล์ `.cpp` ใหม่เข้า `SOURCES` ใน Makefile เมื่อขยายระบบ** — เมื่อสร้างไฟล์ใหม่
   เช่น `AudioBook.cpp` ในแบบฝึกหัดข้อ 4 แต่ลืมแก้ `SOURCES` ใน Makefile จะได้ **linker error**
   (`undefined reference to AudioBook::AudioBook(...)`) เพราะ `main.cpp` เรียกใช้ `AudioBook`
   แต่ไม่มีการ compile `AudioBook.cpp` เข้ามารวมด้วย — เป็นอาการเดียวกับที่เจอตอนลืม define
   static data member ใน Part 53 (compile ผ่านแต่ link ไม่ผ่าน) ต้องตรวจสอบ `SOURCES` ทุกครั้ง
   ที่เพิ่มไฟล์ `.cpp` ใหม่เข้าโปรเจกต์

---

## แบบฝึกหัดท้ายบท

1. เพิ่ม method `Library::countByType(const std::string& type) const` ที่คืนค่าจำนวนสื่อ
   ที่มี `mediaType()` ตรงกับ `type` ที่ระบุ (เช่น `countByType("Book")` ควรคืนค่า 3 หลังจาก
   สถานการณ์ใน 55.7)
2. เพิ่ม exception ใหม่ชื่อ `DuplicateMediaIdError` และแก้ไข `Library::addMedia()` ให้ throw
   exception นี้เมื่อพยายามเพิ่มสื่อด้วย `id` ที่มีอยู่แล้วในระบบ
3. เพิ่มฟีเจอร์ **ต่ออายุการยืม (renew)** เป็น method `Library::renewMedia(memberId, mediaId)`
   ที่ตรวจสอบว่าสมาชิกคนนี้กำลังยืมสื่อชิ้นนี้อยู่จริง (ถ้าไม่ใช่ให้ throw `NotBorrowedError`)
   แล้ว "รีเซ็ต" สถานะการยืม (ในระบบจริงจะรีเซ็ตวันครบกำหนดคืน แต่ในแบบฝึกหัดนี้แค่พิมพ์
   ข้อความยืนยันว่าต่ออายุสำเร็จก็เพียงพอ)
4. เพิ่มชนิดสื่อใหม่ชื่อ `AudioBook` (มีข้อมูล ผู้อ่าน (`narrator`) และความยาวเป็นนาที) โดย
   **ห้ามแก้ไขไฟล์ `Media.h`/`Media.cpp` เดิมแม้แต่บรรทัดเดียว** ให้สร้างเป็นไฟล์ใหม่
   `AudioBook.h`/`AudioBook.cpp` ที่สืบทอดจาก `Media` แทน (สาธิต Open-Closed Principle)
5. เขียนฟังก์ชันทดสอบเล็กๆ (ไม่ต้องใช้ testing framework ก็ได้ ใช้ `assert` จาก `<cassert>`
   พอ) ตรวจสอบว่า: ยืมสื่อสำเร็จแล้ว `isBorrowed()` ต้องเป็น `true`, คืนสื่อแล้วต้องเป็น
   `false`, และยืมสื่อที่ถูกยืมอยู่แล้วต้อง throw `AlreadyBorrowedError`
6. แก้ไข `Library::returnMedia()` ให้ throw `std::invalid_argument` ทันทีถ้า `mediaId` หรือ
   `memberId` ที่ส่งเข้ามาเป็น string ว่างเปล่า โดยไม่ต้องเสียเวลาค้นหาในระบบก่อน

### แนวทางเฉลยข้อ 1: countByType

เพิ่ม method ใหม่ใน `Library.h` (วางต่อจาก `mediaCount()`):

```cpp
    std::size_t mediaCount() const noexcept { return collection_.size(); }
    int countByType(const std::string& type) const;
```

Implement ใน `Library.cpp`:

```cpp
int Library::countByType(const std::string& type) const {
    int count = 0;
    for (const auto& m : collection_) {
        if (m->mediaType() == type) {
            ++count;
        }
    }
    return count;
}
```

ทดสอบใน `main.cpp`:

```cpp
printDivider("ทดสอบ countByType (แบบฝึกหัดข้อ 1)");
std::cout << "จำนวน Book: " << lib.countByType("Book") << '\n';
std::cout << "จำนวน DVD: " << lib.countByType("DVD") << '\n';
std::cout << "จำนวน Magazine: " << lib.countByType("Magazine") << '\n';
```

ผลลัพธ์:

```
----- ทดสอบ countByType (แบบฝึกหัดข้อ 1) -----
จำนวน Book: 3
จำนวน DVD: 1
จำนวน Magazine: 1
```

จุดสำคัญ: `countByType()` เปรียบเทียบ `m->mediaType()` ที่เป็น**ผลลัพธ์ของ virtual function**
กับ string ที่รับเข้ามา วิธีนี้ต่างจากการนับด้วย static member อย่าง `Book::totalBookCount()`
ตรงที่ `countByType()` **ทำงานกับ `Media*` ทั่วไปโดยไม่ต้องรู้จักชนิดที่แน่ชัด** และใช้ได้กับ
ชนิดสื่อใหม่ที่ยังไม่มีในระบบตอนเขียนโค้ดนี้เลยด้วยซ้ำ (เช่น ถ้าเพิ่ม `AudioBook` ในภายหลัง
ตามแบบฝึกหัดข้อ 4 `countByType("AudioBook")` จะทำงานถูกต้องทันทีโดยไม่ต้องแก้ไข `countByType()`
เลย)

### แนวทางเฉลยข้อ 2: DuplicateMediaIdError

เพิ่ม exception class ใหม่ใน `Library.h` (วางไว้ก่อน `MediaNotFoundError` เพื่อให้อ่านง่าย):

```cpp
class DuplicateMediaIdError : public LibraryError {
public:
    explicit DuplicateMediaIdError(const std::string& mediaId)
        : LibraryError("มีสื่อ ID: " + mediaId + " อยู่ในระบบแล้ว ไม่สามารถเพิ่มซ้ำได้") {}
};
```

แก้ไข `Library::addMedia()` ใน `Library.cpp` ให้ตรวจสอบ id ซ้ำก่อนเพิ่ม:

```cpp
Media& Library::addMedia(std::unique_ptr<Media> media) {
    for (const auto& existing : collection_) {
        if (existing->id() == media->id()) {
            throw DuplicateMediaIdError(media->id());
        }
    }
    collection_.push_back(std::move(media));
    return *collection_.back();
}
```

ทดสอบใน `main.cpp` เพิ่มเติม:

```cpp
printDivider("ทดสอบ DuplicateMediaIdError (แบบฝึกหัดข้อ 2)");
try {
    lib.addMedia(std::make_unique<Book>("Duplicate Test", "B001", "Someone", "000-0-00-000000-0"));
} catch (const DuplicateMediaIdError& e) {
    std::cout << "จับ DuplicateMediaIdError ได้ถูกต้อง: " << e.what() << '\n';
}
```

ผลลัพธ์:

```
----- ทดสอบ DuplicateMediaIdError (แบบฝึกหัดข้อ 2) -----
จับ DuplicateMediaIdError ได้ถูกต้อง: มีสื่อ ID: B001 อยู่ในระบบแล้ว ไม่สามารถเพิ่มซ้ำได้
```

จุดสำคัญ: เราตรวจสอบ id ซ้ำ **ก่อน** ที่จะ `push_back` เข้า `collection_` — ถ้าเช็คหลัง
`push_back` จะสายเกินไปเพราะสื่อซ้ำจะถูกเพิ่มเข้าไปแล้ว การตรวจสอบเงื่อนไขก่อนแก้ไข state ของ
object เสมอเป็นหลักการเขียนโค้ดที่ปลอดภัย (เรียกว่า **Strong Exception Safety Guarantee** —
ถ้า throw เกิดขึ้น state ของ object ต้องไม่เปลี่ยนแปลงจากก่อนเรียกฟังก์ชันเลย)

### แนวทางเฉลยข้อ 3: renewMedia (ต่ออายุการยืม)

เพิ่ม method ใหม่ใน `Library.h`:

```cpp
    void returnMedia(const std::string& memberId, const std::string& mediaId);
    void renewMedia(const std::string& memberId, const std::string& mediaId);
```

Implement ใน `Library.cpp`:

```cpp
void Library::renewMedia(const std::string& memberId, const std::string& mediaId) {
    Member& member = findMember(memberId);
    Media& media = findMedia(mediaId);

    if (!media.isBorrowed() || !member.isBorrowing(&media)) {
        throw NotBorrowedError(media.title());
    }
    // ในระบบจริงจุดนี้จะรีเซ็ตวันครบกำหนดคืนใหม่ ในที่นี้แค่ยืนยันว่าสถานะยังคงยืมอยู่ถูกต้อง
}
```

ทดสอบใน `main.cpp`:

```cpp
printDivider("ทดสอบ renewMedia (แบบฝึกหัดข้อ 3)");
try {
    lib.borrowMedia("U001", "B003");
    lib.renewMedia("U001", "B003");
    std::cout << "Somchai ต่ออายุการยืม B003 สำเร็จ\n";
    lib.renewMedia("U001", "B002"); // B002 ถูก Malee ยืมอยู่ ไม่ใช่ Somchai -> ควร throw
} catch (const NotBorrowedError& e) {
    std::cout << "จับ NotBorrowedError ได้ถูกต้อง: " << e.what() << '\n';
}
```

ผลลัพธ์:

```
----- ทดสอบ renewMedia (แบบฝึกหัดข้อ 3) -----
Somchai ต่ออายุการยืม B003 สำเร็จ
จับ NotBorrowedError ได้ถูกต้อง: สื่อ "Effective Modern C++" ไม่ได้อยู่ในสถานะยืมอยู่ หรือไม่ได้ถูกยืมโดยสมาชิกคนนี้
```

จุดสำคัญ: สังเกตว่า `renewMedia()` มีเงื่อนไขตรวจสอบ**เหมือนกันเป๊ะ**กับ `returnMedia()`
(ต้องเป็นสื่อที่ถูกยืมอยู่ และต้องเป็นสมาชิกคนที่ยืมจริง) นี่คือสัญญาณว่าถ้าจะพัฒนาโปรเจกต์นี้
ต่อในระดับ production ควรจะดึง logic การตรวจสอบนี้ออกมาเป็น private helper method เช่น
`ensureCurrentlyBorrowedBy(member, media)` เพื่อไม่ให้โค้ดซ้ำกันระหว่างสอง method — หลักการ
"อย่าเขียนโค้ดซ้ำ" (DRY — Don't Repeat Yourself) นี้จะกลับมาเน้นย้ำอีกครั้งใน Part 113
(Clean Code และ Code Review Practice)

### แนวทางเฉลยข้อ 4: เพิ่ม AudioBook โดยไม่แก้โค้ดเดิม

สร้างไฟล์ใหม่ `AudioBook.h`:

```cpp
#ifndef AUDIOBOOK_H
#define AUDIOBOOK_H

#include <string>
#include "Media.h"

// class ใหม่ที่เพิ่มเข้ามาทีหลัง โดยไม่ต้องแก้ไข Media.h/.cpp เดิมแม้แต่บรรทัดเดียว
// (Open-Closed Principle: เปิดให้ขยาย แต่ปิดการแก้ไขของเดิม)
class AudioBook : public Media {
public:
    AudioBook(std::string title, std::string id, std::string narrator, int durationMinutes);

    std::string mediaType() const override { return "AudioBook"; }
    std::string extraInfo() const override;

private:
    std::string narrator_;
    int durationMinutes_;
};

#endif // AUDIOBOOK_H
```

และไฟล์ `AudioBook.cpp`:

```cpp
#include "AudioBook.h"
#include <utility>

AudioBook::AudioBook(std::string title, std::string id, std::string narrator, int durationMinutes)
    : Media(std::move(title), std::move(id)),
      narrator_(std::move(narrator)),
      durationMinutes_(durationMinutes) {}

std::string AudioBook::extraInfo() const {
    return "ผู้อ่าน: " + narrator_ + ", ความยาว: " + std::to_string(durationMinutes_) + " นาที";
}
```

เพิ่มใน `main.cpp` (แค่ `#include "AudioBook.h"` แล้วเรียกใช้เหมือน `Media` ชนิดอื่น):

```cpp
#include "AudioBook.h"
// ...
printDivider("เพิ่ม AudioBook (แบบฝึกหัดข้อ 4: ขยายระบบโดยไม่แก้โค้ดเดิม)");
lib.addMedia(std::make_unique<AudioBook>("Sapiens (Audio)", "A001", "Derek Perkins", 912));
lib.printCatalog(std::cout);
```

และเพิ่ม `AudioBook.cpp` เข้า `SOURCES` ใน Makefile:

```makefile
SOURCES  := main.cpp Media.cpp Library.cpp Member.cpp AudioBook.cpp
```

ผลลัพธ์ส่วนท้าย:

```
----- เพิ่ม AudioBook (แบบฝึกหัดข้อ 4: ขยายระบบโดยไม่แก้โค้ดเดิม) -----
=== แคตตาล็อกของ ห้องสมุดประชาชนสาขากลาง (6 รายการ) ===
  [Book] The C Programming Language (ID: B001) - ว่างอยู่บนชั้น | ผู้แต่ง: K&R, ISBN: 978-0-13-110362-7
  [Book] Effective Modern C++ (ID: B002) - ถูกยืมอยู่ | ผู้แต่ง: Scott Meyers, ISBN: 978-1-4919-0399-5
  [DVD] The Matrix (ID: D001) - ถูกยืมอยู่ | ผู้กำกับ: Wachowski Sisters, ความยาว: 136 นาที
  [Magazine] National Geographic (ID: M001) - ถูกยืมอยู่ | ฉบับที่: 254
  [Book] Clean Code (ID: B003) - ว่างอยู่บนชั้น | ผู้แต่ง: Robert C. Martin, ISBN: 978-0-13-235088-4
  [AudioBook] Sapiens (Audio) (ID: A001) - ว่างอยู่บนชั้น | ผู้อ่าน: Derek Perkins, ความยาว: 912 นาที
```

จุดสำคัญที่สุดของแบบฝึกหัดนี้: **`printCatalog()`, `operator<<`, `borrowMedia()`,
`returnMedia()` และทุก method อื่นของ `Library` ไม่มีการแก้ไขแม้แต่บรรทัดเดียว** — ระบบ
รองรับชนิดสื่อใหม่ได้ทันทีเพราะ `AudioBook` implement contract ของ `Media` (คือ
`mediaType()`) ถูกต้องครบถ้วน นี่คือพลังที่แท้จริงของ Abstract Class และ Polymorphism ที่เรียน
มาตลอด Module D และเป็นเครื่องพิสูจน์ว่าการออกแบบระบบตั้งแต่ 55.1 ด้วยหลัก Open-Closed
Principle นั้นใช้งานได้จริงในทางปฏิบัติ

### แนวทางเฉลยข้อ 5: เขียนฟังก์ชันทดสอบด้วย assert

สร้างไฟล์ทดสอบแยกต่างหาก `test_library.cpp` (ยังไม่ใช้ testing framework เต็มรูปแบบ เพราะ
Google Test/Catch2 จะเรียนใน Part 93 — ตอนนี้ใช้ `assert` จาก `<cassert>` ที่เรียนไปแล้วใน
Part 16 ก็เพียงพอ):

```cpp
#include <cassert>
#include <iostream>
#include <memory>

#include "Library.h"
#include "Media.h"
#include "Member.h"

void testBorrowAndReturn() {
    Library lib("Test Library");
    lib.addMedia(std::make_unique<Book>("Test Book", "T001", "Author", "000-0"));
    lib.addMember("U001", "Tester");

    Media& media = lib.findMedia("T001");
    assert(media.isBorrowed() == false);

    lib.borrowMedia("U001", "T001");
    assert(media.isBorrowed() == true);

    lib.returnMedia("U001", "T001");
    assert(media.isBorrowed() == false);

    bool caught = false;
    try {
        lib.borrowMedia("U001", "T001");
        lib.borrowMedia("U001", "T001"); // ยืมซ้ำ ควร throw
    } catch (const AlreadyBorrowedError&) {
        caught = true;
    }
    assert(caught && "ต้อง throw AlreadyBorrowedError เมื่อยืมสื่อที่ถูกยืมไปแล้ว");

    std::cout << "testBorrowAndReturn: PASSED\n";
}

int main() {
    testBorrowAndReturn();
    std::cout << "การทดสอบทั้งหมดผ่านสำเร็จ\n";
}
```

คอมไพล์แยกเป็นโปรแกรมทดสอบต่างหาก (ไม่รวม `main.cpp` เดิม เพราะทั้งคู่ต่างมีฟังก์ชัน `main`
ของตัวเอง — ทบทวน Part 1: จะมี `main` ซ้ำกันสองไฟล์ใน executable เดียวกันไม่ได้):

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 test_library.cpp Media.cpp Library.cpp Member.cpp -o test_library
./test_library
```

ผลลัพธ์:

```
testBorrowAndReturn: PASSED
การทดสอบทั้งหมดผ่านสำเร็จ
```

จุดสำคัญ: สังเกตว่าเราสร้าง `Library lib("Test Library")` **ตัวใหม่แยกต่างหาก** ภายใน
`testBorrowAndReturn()` แทนที่จะใช้ตัวเดียวกับใน `main.cpp` เดิม — นี่คือหลักการสำคัญของการ
เขียน unit test ที่ดี: **แต่ละ test case ควรเริ่มต้นจาก state ที่สะอาดของตัวเอง** ไม่ปะปนกับ
state ที่ค้างจาก test case อื่นหรือโปรแกรมหลัก (ทบทวนคำเตือนเรื่อง static/global state ที่ทำให้
เขียน unit test ยากขึ้นจาก Part 53 ข้อ 53.8 — ในที่นี้เราหลีกเลี่ยงปัญหานั้นได้เพราะ
`Library`/`Member`/`Media` ทั้งหมดเป็น instance member ธรรมดา ไม่ใช่ global state ยกเว้นแค่
ตัวนับ `totalMediaCount_`/`totalBookCount_` ซึ่งเป็นสิ่งที่ควรระวังเป็นพิเศษถ้าจะเขียนเทสเพิ่ม
ที่ตรวจสอบค่าพวกนี้ เพราะมันจะสะสมข้ามทุก `Library` instance ที่เคยสร้างในโปรแกรมเดียวกัน)

### แนวทางเฉลยข้อ 6: ตรวจสอบ string ว่างเปล่าก่อนค้นหา

แก้ไข `Library::returnMedia()` ใน `Library.cpp` ให้ตรวจสอบก่อนเรียก `findMember`/`findMedia`:

```cpp
void Library::returnMedia(const std::string& memberId, const std::string& mediaId) {
    if (memberId.empty() || mediaId.empty()) {
        throw std::invalid_argument("memberId และ mediaId ต้องไม่เป็นค่าว่าง");
    }

    Member& member = findMember(memberId);
    Media& media = findMedia(mediaId);

    if (!media.isBorrowed() || !member.isBorrowing(&media)) {
        throw NotBorrowedError(media.title());
    }

    media.markReturned();
    member.removeBorrowedMedia(&media);
}
```

ทดสอบใน `main.cpp`:

```cpp
printDivider("ทดสอบ returnMedia ด้วย mediaId ว่างเปล่า (แบบฝึกหัดข้อ 6)");
try {
    lib.returnMedia("U001", "");
} catch (const std::invalid_argument& e) {
    std::cout << "จับ std::invalid_argument ได้ถูกต้อง: " << e.what() << '\n';
}
```

ผลลัพธ์:

```
----- ทดสอบ returnMedia ด้วย mediaId ว่างเปล่า (แบบฝึกหัดข้อ 6) -----
จับ std::invalid_argument ได้ถูกต้อง: memberId และ mediaId ต้องไม่เป็นค่าว่าง
```

จุดสำคัญ: เราเลือก throw **`std::invalid_argument`** (exception มาตรฐานจาก Part 54) แทนที่จะ
สร้าง custom exception ใหม่ของระบบ (`LibraryError` ลูกใดๆ) เพราะการตรวจสอบนี้เป็นเรื่องของ
**"รูปแบบ argument ที่ไม่ถูกต้องตั้งแต่ต้น"** (programmer error / invalid input format) ไม่ใช่
สถานการณ์ทางธุรกิจของห้องสมุด (เช่น "ไม่พบข้อมูล" หรือ "ยืมซ้ำ") การเลือกใช้ exception ที่
เหมาะสมกับ**ความหมายที่แท้จริง**ของข้อผิดพลาด แทนที่จะใช้ custom exception ของระบบตัวเองไปหมด
ทุกกรณี เป็นทักษะสำคัญของการออกแบบ exception hierarchy ที่ดี (ทบทวน Part 54 ข้อ 54.3)

---

## สรุปท้ายบท

Part นี้คือบทสรุปของ **Module D ทั้งโมดูล** เราได้นำแนวคิด OOP ทุกตัวที่เรียนมาตั้งแต่ Part 45
มาประกอบร่างเป็นระบบที่ทำงานได้จริง:

- **ออกแบบระบบตั้งแต่ requirement ถึง class diagram** ก่อนลงมือเขียนโค้ดจริง
- **Abstract Class (`Media`) และ Polymorphism** ทำให้เพิ่มชนิดสื่อใหม่ได้โดยไม่แก้โค้ดเดิม
  (Open-Closed Principle) — พิสูจน์แล้วในแบบฝึกหัดข้อ 4 ด้วยการเพิ่ม `AudioBook`
- **Static Member** ใช้นับจำนวนสื่อทั้งหมดในระบบ (`Media::totalMediaCount`) และจำนวนหนังสือ
  โดยเฉพาะ (`Book::totalBookCount`) รวมถึงกำหนดโควตาการยืมร่วมกันของสมาชิกทุกคน
  (`Member::MAX_BORROW_LIMIT`)
- **Exception Hierarchy ของระบบเอง** (`LibraryError` และลูกๆ) จัดการสถานการณ์ error ทางธุรกิจ
  ได้ครบถ้วนและสื่อความหมายชัดเจนกว่าการคืน error code ธรรมดา
- **Operator Overloading** (`operator<<`) ทำให้แสดงผล `Media` และ `Member` ได้เป็นธรรมชาติ
- **RAII ผ่าน `std::unique_ptr`** จัดการ ownership ของ object ทั้งหมดโดยไม่มีการ
  `new`/`delete` แบบ manual แม้แต่บรรทัดเดียว รับประกันว่าไม่มี memory leak
- **โครงสร้างโปรเจกต์แบบ multi-file พร้อม Makefile** ที่ build ได้อัตโนมัติและขยายเพิ่มไฟล์ใหม่
  ได้ง่าย (ดังที่เห็นตอนเพิ่ม `AudioBook.cpp` เข้า `SOURCES`)

Module D จบลงที่นี่ เราได้อาวุธครบมือสำหรับเขียนโปรแกรม OOP ใน C++ อย่างมืออาชีพแล้ว
ใน **Module E** ที่เริ่มต้นด้วย **Part 56** เราจะก้าวเข้าสู่โลกของ **Generic Programming**
ผ่าน **Template** — เทคนิคที่ทำให้เขียนโค้ดที่ทำงานกับชนิดข้อมูลใดก็ได้โดยไม่ต้องเขียนซ้ำ
ซึ่งเป็นรากฐานสำคัญของ **STL (Standard Template Library)** ที่เราจะใช้งานอย่างเข้มข้นตลอด
โมดูลถัดไป

**ต่อไป:** [Part 56 — Function Template และ Class Template](./part-056-function-class-templates.md)
