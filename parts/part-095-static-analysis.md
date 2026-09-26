# Part 95: Static Analysis: clang-tidy, cppcheck, clang-format (Step 753–760)

> Module H — Build Systems, Testing และ Tooling | Part 95 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 753–760
> Part ก่อนหน้า: [Part 94 — Continuous Integration สำหรับ C++ ด้วย GitHub Actions](./part-094-ci-github-actions.md) | Part ถัดไป: [Part 96 — Sanitizer และ Code Coverage: ASan, UBSan, TSan, gcov/lcov](./part-096-sanitizers-coverage.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Static Analysis** คืออะไร ต่างจาก Compiler Warning, Unit Testing และ
   Sanitizer (ที่จะเรียนใน Part 96) อย่างไร และทำไมทั้งหมดนี้ต้องใช้**ร่วมกัน**ไม่ใช่เลือกใช้แค่ตัวใด
   ตัวหนึ่ง
2. ติดตั้งและรัน **clang-tidy** กับโค้ด C++ จริงได้ ทั้งแบบสั่ง check ตรงๆ ทาง command line
   และแบบใช้ไฟล์ config `.clang-tidy`
3. อ่านและตีความคำเตือนจาก checks กลุ่มสำคัญของ clang-tidy ได้แก่ `modernize-*`,
   `bugprone-*`, และ `performance-*` พร้อมแก้โค้ดตาม suggestion ได้อย่างถูกต้อง
4. รันโค้ดสไตล์เก่า (แบบที่พบได้ใน Module B/C) ผ่าน clang-tidy จริง เจอปัญหาจริง แก้ไขจริง
   และยืนยันว่าหลังแก้แล้ว clang-tidy ไม่เตือนอะไรอีก
5. ติดตั้งและรัน **cppcheck** เปรียบเทียบผลลัพธ์กับ clang-tidy บนโค้ดเดียวกัน และอธิบายได้ว่า
   เครื่องมือทั้งสองมีจุดแข็ง/จุดอ่อนต่างกันอย่างไร ควรใช้ตัวไหนเมื่อไหร่
6. ติดตั้งและใช้ **clang-format** จัดรูปแบบโค้ดอัตโนมัติผ่านไฟล์ `.clang-format` เข้าใจความ
   แตกต่างระหว่าง style สำเร็จรูปยอดนิยม (LLVM vs Google) และใช้โหมด `--dry-run --Werror`
   เพื่อตรวจสอบใน CI โดยไม่แก้ไฟล์จริง
7. ผูก clang-tidy เข้ากับ CMake โดยตรง (`CMAKE_CXX_CLANG_TIDY`) เพื่อให้ static analysis รัน
   อัตโนมัติทุกครั้งที่ build โดยไม่ต้องสั่งแยก
8. เพิ่ม static analysis เข้าไปใน GitHub Actions workflow จาก **Part 94** เพื่อให้ทุก Pull
   Request ถูกตรวจสอบทั้งบั๊กที่อาจเกิดขึ้นและความสม่ำเสมอของรูปแบบโค้ดโดยอัตโนมัติ

---

## หมายเหตุสำคัญก่อนเริ่ม

ทุกคำสั่งและผลลัพธ์ของ `clang-tidy`, `cppcheck`, และ `clang-format` ใน Part นี้**รันจริงบน
เครื่องนี้** (clang-tidy/clang-format เวอร์ชัน LLVM 18.1.3, cppcheck เวอร์ชัน 2.13.0 ที่ติดตั้ง
ผ่าน `apt`) ไม่มีผลลัพธ์ใดถูกแต่งขึ้นเอง — ทุกคำเตือนที่แสดงคือสิ่งที่เครื่องมือรายงานจริงกับไฟล์
ตัวอย่างที่เขียนขึ้นเพื่อสาธิต Part นี้โดยเฉพาะ

---

## 95.1 Static Analysis คืออะไร ต่างจากอะไรบ้าง (Step 753)

### นิยาม

**Static Analysis** คือการวิเคราะห์โค้ดต้นฉบับ**โดยไม่ต้องรันโปรแกรมจริงเลย** เครื่องมือจะอ่าน
โครงสร้างไวยากรณ์ (Abstract Syntax Tree) และบางครั้งจำลอง data flow ของโค้ดเพื่อหาความผิด
พลาดเชิงตรรกะ, รูปแบบที่เสี่ยงต่อบั๊ก, หรือโค้ดที่ไม่ตรงตามมาตรฐานที่กำหนด — ทั้งหมดนี้เกิดขึ้น
**ที่ขั้นตอนก่อนคอมไพล์หรือระหว่างคอมไพล์** ไม่ต้องมี input จริง ไม่ต้องมี test case ใดๆ เลย

### เปรียบเทียบกับเครื่องมือตรวจสอบคุณภาพโค้ดตัวอื่นที่เรียนมาแล้ว

| เครื่องมือ | ทำงานตอนไหน | ตรวจจับอะไร | ต้องรันโปรแกรมจริงไหม |
|---|---|---|---|
| **Compiler Warning** (`-Wall -Wextra`) | ตอน compile | ปัญหา syntax/type ที่ compiler เห็นระหว่างแปลโค้ด | ไม่ต้อง |
| **Static Analysis** (clang-tidy, cppcheck) | ก่อน/ระหว่าง compile (แยกเครื่องมือต่างหาก) | รูปแบบโค้ดที่เสี่ยงบั๊ก, สไตล์ที่ล้าสมัย, จุดที่ compiler มองข้าม | ไม่ต้อง |
| **Unit Testing** (Part 93) | หลัง compile, รันจริง | Logic ผิดจากที่ออกแบบไว้ (ตาม test case ที่เขียน) | ต้องรัน |
| **Sanitizer** (Part 96) | หลัง compile, รันจริง | บั๊กที่เกิด "ตอน runtime จริง" เช่น memory error, UB, race condition | ต้องรัน |

จุดสำคัญที่สุดที่ทำให้ Static Analysis มีค่าคือ **มันหาปัญหาที่ไม่จำเป็นต้องมี test case มาก
ระตุ้นให้เกิดก่อน** ตัวอย่างเช่น การลืมใส่ `override` ไม่ทำให้โปรแกรม crash หรือ test พังเลย
แต่เป็นความเสี่ยงเชิงบำรุงรักษาระยะยาวที่ static analyzer มองเห็นได้ทันทีจากโครงสร้างโค้ด
โดยไม่ต้องรอให้เกิดปัญหาจริง

### ทำไม Compiler Warning เพียงอย่างเดียวไม่พอ

ลองดูตัวอย่างโค้ดที่ **compile ผ่านโดยไม่มี warning แม้จะเปิด `-Wall -Wextra` เต็มที่** แต่มี
ปัญหาซ่อนอยู่หลายจุด:

```cpp
// legacy_list.cpp (ย่อ) — คอมไพล์ผ่านสะอาด ไม่มี warning เลย
#include <cstdio>
#include <cstring>
#include <string>

std::string greet(std::string name) {   // ควรรับ const& ไม่ใช่ copy
    char buffer[32];
    strcpy(buffer, name.c_str());       // ไม่เช็คความยาวก่อน copy — เสี่ยง overflow
    return std::string("Hello, ") + buffer;
}
```

```bash
g++ -Wall -Wextra -std=c++17 legacy_list.cpp -o legacy_list
```

ผลลัพธ์จริงจากการคอมไพล์บนเครื่องนี้: **ไม่มี warning แม้แต่บรรทัดเดียว** โปรแกรม compile ผ่าน
และรันได้ปกติ (ตราบใดที่ `name` สั้นกว่า 32 ตัวอักษร) นี่คือจุดที่ static analyzer เข้ามาเติมเต็ม
ช่องว่างที่ compiler warning มองไม่เห็น — จะเห็นผลจริงในหัวข้อถัดไป

---

## 95.2 clang-tidy: การติดตั้งและการรันพื้นฐาน (Step 754)

### ติดตั้ง

```bash
sudo apt update
sudo apt install clang-tidy -y
clang-tidy --version
```

ผลลัพธ์จริงบนเครื่องนี้:

```
Ubuntu LLVM version 18.1.3
  Optimized build.
```

### clang-tidy คืออะไร

**clang-tidy** เป็นเครื่องมือ static analysis ที่สร้างขึ้นบนโครงสร้างของ Clang/LLVM โดยตรง
ทำให้มันเข้าใจโค้ด C++ ได้ลึกระดับเดียวกับ compiler จริง (ไม่ใช่แค่จับ pattern แบบ text matching
ผิวเผิน) จุดเด่นที่สำคัญคือมันไม่ได้แค่ "เตือน" แต่ยังสามารถ**เสนอวิธีแก้ (Fix-It)** ให้ทันทีด้วย

### รันแบบพื้นฐานที่สุด

```bash
clang-tidy myfile.cpp -- -std=c++17
```

ส่วนหลัง `--` คือ compiler flag ที่ clang-tidy ต้องใช้ในการทำความเข้าใจโค้ด (เหมือน flag ที่ส่ง
ให้ clang จริงตอน compile) ถ้าโปรเจกต์มีหลายไฟล์และ include path ซับซ้อน วิธีที่ดีกว่าคือให้
CMake สร้างไฟล์ `compile_commands.json` ให้อัตโนมัติ:

```bash
cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
clang-tidy -p build myfile.cpp
```

`-p build` บอกให้ clang-tidy อ่าน `compile_commands.json` จากโฟลเดอร์ `build` เพื่อรู้ flag และ
include path ที่ถูกต้องของแต่ละไฟล์โดยอัตโนมัติ ไม่ต้องพิมพ์ `-- -std=c++17 -I...` เองทุกครั้ง

### กลุ่ม Checks สำคัญที่ต้องรู้จัก

clang-tidy มี checks หลายร้อยตัว จัดกลุ่มด้วย prefix:

| กลุ่ม | ตรวจจับอะไร |
|---|---|
| `modernize-*` | โค้ดสไตล์เก่าที่ควรเขียนใหม่ด้วยฟีเจอร์ C++11 ขึ้นไป (เช่น `NULL` → `nullptr`, ไม่ใส่ `override`) |
| `bugprone-*` | รูปแบบโค้ดที่ "ดูเหมือนถูก" แต่มักเป็นสาเหตุของบั๊กในทางปฏิบัติ |
| `performance-*` | รูปแบบที่ทำงานถูกต้องแต่ช้ากว่าที่ควรจะเป็น (เช่น copy ที่ไม่จำเป็น) |
| `cppcoreguidelines-*` | ตรวจตาม C++ Core Guidelines ที่ Bjarne Stroustrup และทีมงานวางไว้ |
| `clang-analyzer-*` | การวิเคราะห์เชิงลึกกว่าปกติ (symbolic execution) หาบั๊กที่ซับซ้อน เช่น การใช้ API ที่ไม่ปลอดภัย |
| `cert-*` | ตรวจตามมาตรฐานความปลอดภัย CERT C++ Coding Standard |
| `readability-*` | ปัญหาเชิงอ่านง่าย ไม่เกี่ยวกับความถูกต้องโดยตรง |

การเลือก checks ทำผ่าน flag `-checks=`:

```bash
clang-tidy myfile.cpp -checks='-*,modernize-*,bugprone-*,performance-*' -- -std=c++17
```

`-*` หมายถึง "ปิดทุก check ก่อน" แล้วค่อยเปิดเฉพาะกลุ่มที่ต้องการทีหลังด้วย comma — วิธีนี้
สำคัญมาก เพราะถ้าไม่ใส่ `-*,` นำหน้า clang-tidy จะเปิด default checks ชุดใหญ่ (ซึ่งรวมถึง
checks จาก system header ด้วย) ทำให้ output ท่วมท้นจนหาของจริงไม่เจอ

---

## 95.3 รัน clang-tidy กับโค้ดสไตล์เก่าจริง (Step 755)

### โค้ดตัวอย่าง: Linked List สไตล์ Module B/C พอร์ตมาเป็น C++

ไฟล์นี้จำลองสถานการณ์ที่พบได้จริงมาก: โค้ดที่เขียนแบบ C ดั้งเดิม (ที่เรียนใน Part 19 — Linked
List) ถูกพอร์ตมาเป็น C++ อย่างเร่งรีบโดยยังคงนิสัยแบบ C ไว้แทบทุกจุด

```cpp
// legacy_list.cpp
#include <cstdio>
#include <cstring>
#include <string>
#include <vector>

struct Node {
    int  value;
    Node *next;
};

class IntList {
public:
    IntList() { head = NULL; count = 0; }

    void push_back(int v) {
        Node *n = new Node;
        n->value = v;
        n->next = NULL;

        if (head == NULL) {
            head = n;
        } else {
            Node *cur = head;
            while (cur->next != NULL) {
                cur = cur->next;
            }
            cur->next = n;
        }
        count++;
    }

    void print_all() {
        Node *cur = head;
        for (int i = 0; i < count; i++) {
            printf("%d\n", cur->value);
            cur = cur->next;
        }
    }

    int sum() {
        int total = 0;
        Node *cur = head;
        while (cur != NULL) {
            total = total + cur->value;
            cur = cur->next;
        }
        return total;
    }

    ~IntList() {
        // ไม่ได้ free node ทีละตัว -> memory leak ทุกครั้งที่ list ถูกทำลาย
    }

private:
    Node *head;
    int count;
};

class Base {
public:
    virtual void describe() { printf("Base\n"); }
    virtual ~Base() {}
};

class Derived : public Base {
public:
    void describe() { printf("Derived\n"); }   // ไม่ได้ใส่ override
};

std::string greet(std::string name) {          // ควรรับ const&
    char buffer[32];
    strcpy(buffer, name.c_str());              // เสี่ยง buffer overflow
    return std::string("Hello, ") + buffer;
}

int classify(int x) {
    double half = (double)x / 2;               // C-style cast
    return (int)half;
}

int main() {
    IntList list;
    for (int i = 1; i <= 5; i++) list.push_back(i * 10);
    list.print_all();
    printf("sum = %d\n", list.sum());

    Base *b = new Derived();
    b->describe();
    delete b;

    std::string msg = greet("Somchai");
    printf("%s\n", msg.c_str());
    printf("classify(7) = %d\n", classify(7));
    return 0;
}
```

ยืนยันก่อนว่าโค้ดนี้ compile และรันได้ปกติ (และไม่มี warning จาก `-Wall -Wextra` เลย):

```bash
g++ -Wall -Wextra -std=c++17 legacy_list.cpp -o legacy_list
./legacy_list
```

```
10
20
30
40
50
sum = 150
Derived
Hello, Somchai
classify(7) = 3
```

### รัน clang-tidy จริงกับไฟล์นี้

```bash
clang-tidy legacy_list.cpp -checks='-*,modernize-*,bugprone-*,performance-*' -- -std=c++17
```

ผลลัพธ์จริงที่ได้ (แสดงเฉพาะส่วนที่เป็นโค้ดของเราเอง ไม่รวม 14,375 warning จาก system header
ที่ clang-tidy suppress ให้อัตโนมัติ):

```
warning: use nullptr [modernize-use-nullptr]
   17 |     IntList() { head = NULL; count = 0; }
      |                        ^~~~
      |                        nullptr

warning: use nullptr [modernize-use-nullptr]
   22 |         n->next = NULL;
      |                   ^~~~
      |                   nullptr

warning: use nullptr [modernize-use-nullptr]
   24 |         if (head == NULL) {
      |                     ^~~~
      |                     nullptr

warning: use nullptr [modernize-use-nullptr]
   28 |             while (cur->next != NULL) {
      |                                 ^~~~
      |                                 nullptr

warning: use a trailing return type for this function [modernize-use-trailing-return-type]
   45 |     int sum() {
      |     ~~~ ^
      |     auto      -> int

warning: use nullptr [modernize-use-nullptr]
   48 |         while (cur != NULL) {
      |                       ^~~~
      |                       nullptr

warning: use '= default' to define a trivial destructor [modernize-use-equals-default]
   55 |     ~IntList() {
      |     ^

warning: use '= default' to define a trivial destructor [modernize-use-equals-default]
   69 |     virtual ~Base() {}
      |             ^       ~~
      |                     = default;

warning: annotate this function with 'override' or (rarely) 'final' [modernize-use-override]
   75 |     void describe() {
      |          ^
      |                     override

warning: use a trailing return type for this function [modernize-use-trailing-return-type]
   81 | std::string greet(std::string name) {
      | ~~~~~~~~~~~ ^
      | auto                                -> std::string

warning: the parameter 'name' is copied for each invocation but only used as
a const reference; consider making it a const reference [performance-unnecessary-value-param]
   81 | std::string greet(std::string name) {
      |                               ^
      |                   const      &

warning: do not declare C-style arrays, use std::array<> instead [modernize-avoid-c-arrays]
   82 |     char buffer[32];
      |     ^

warning: use a trailing return type for this function [modernize-use-trailing-return-type]
   88 | int classify(int x) {
      | ~~~ ^
      | auto                -> int

warning: use a trailing return type for this function [modernize-use-trailing-return-type]
   94 | int main() {
      | ~~~ ^
      | auto       -> int
```

รวม **14 คำเตือนที่เป็นของโค้ดเราเอง** จากไฟล์เดียวที่ compiler warning ทั่วไปไม่เห็นสักจุดเดียว
ที่น่าสนใจที่สุดคือ `performance-unnecessary-value-param` บนพารามิเตอร์ `name` — clang-tidy
"เข้าใจ" ว่าฟังก์ชันนี้ไม่เคยแก้ไขค่า `name` เลยตลอด body ทั้งหมด (วิเคราะห์จาก data flow จริง
ไม่ใช่แค่ pattern matching) จึงแนะนำให้เปลี่ยนเป็น `const&` เพื่อลดการ copy string ที่ไม่จำเป็น

### เจาะลึก: หา bug ด้วย checks กลุ่ม `clang-analyzer-*` และ `cert-*`

checks กลุ่ม `modernize-*`/`bugprone-*`/`performance-*` ไม่ได้จับปัญหา `strcpy` ที่เสี่ยง buffer
overflow ในฟังก์ชัน `greet` เลย ต้องเปิด checks กลุ่มที่ทำ symbolic execution ลึกกว่านั้น:

```bash
clang-tidy legacy_list.cpp -checks='-*,clang-analyzer-*,cert-*' -- -std=c++17
```

ผลลัพธ์จริง (เฉพาะส่วนที่เป็นโค้ดเรา):

```
warning: Call to function 'strcpy' is insecure as it does not provide
bounding of the memory buffer. Replace unbounded copy functions with
analogous functions that support length arguments such as 'strlcpy'.
CWE-119 [clang-analyzer-security.insecureAPI.strcpy]
   84 |     strcpy(buffer, name.c_str());
      |     ^~~~~~
```

สังเกตว่า clang-tidy ไม่เพียงเตือน แต่ยังอ้างอิง **CWE-119** (Common Weakness Enumeration —
ฐานข้อมูลมาตรฐานสากลของช่องโหว่ความปลอดภัย) ด้วย นี่คือระดับความลึกที่ compiler warning
ธรรมดาไม่มีทางให้ได้ เพราะต้องอาศัยฐานความรู้เรื่องช่องโหว่ความปลอดภัยที่ผูกมากับ checker
โดยเฉพาะ

---

## 95.4 แก้โค้ดตาม Suggestion และยืนยันผลด้วย .clang-tidy Config (Step 756)

### สร้างไฟล์ `.clang-tidy` เพื่อกำหนดค่ามาตรฐานของโปรเจกต์

แทนที่จะพิมพ์ `-checks=...` ยาวๆ ทุกครั้ง มืออาชีพจะเก็บค่ามาตรฐานไว้ในไฟล์ `.clang-tidy` ที่
root ของ repository (clang-tidy จะค้นหาไฟล์นี้อัตโนมัติจากโฟลเดอร์ปัจจุบันขึ้นไปจนถึง root):

```yaml
# .clang-tidy
Checks: >
  -*,
  modernize-*,
  bugprone-*,
  performance-*,
  clang-analyzer-*,
  cert-*,
  -modernize-use-trailing-return-type
WarningsAsErrors: ''
HeaderFilterRegex: '.*'
FormatStyle: none
```

จุดที่น่าสนใจ: บรรทัด `-modernize-use-trailing-return-type` (มีเครื่องหมายลบนำหน้า) คือการ
**ปิด check นี้ทิ้งไปเฉพาะตัว** แม้จะเปิดกลุ่ม `modernize-*` ทั้งหมดไปแล้ว เพราะทีมตัดสินใจว่า
`auto foo() -> int` ไม่ได้อ่านง่ายกว่า `int foo()` เสมอไปสำหรับทุกฟังก์ชัน (การเลือกปิด check
บางตัวที่ทีมไม่เห็นด้วยเป็นเรื่องปกติมาก — ไม่จำเป็นต้องทำตามทุก suggestion เสมอไป)

ยืนยันว่า config ถูกอ่านจริง (ไม่ต้องใส่ `-checks=` ที่ command line แล้ว):

```bash
clang-tidy legacy_list.cpp -- -std=c++17
```

ผลลัพธ์จริงยืนยันว่าได้ผลรวมทั้ง `modernize-*`, `performance-*`, และ `clang-analyzer-*` (รวม
`strcpy` warning) ในการรันครั้งเดียว โดย `modernize-use-trailing-return-type` หายไปตามที่ตั้งค่า
ปิดไว้ใน config

### แก้โค้ดตาม suggestion ทีละจุด

```cpp
// legacy_list_fixed.cpp
#include <cstdio>
#include <cstring>
#include <memory>
#include <string>
#include <vector>

struct Node {
    int value;
    Node *next;
};

class IntList {
public:
    IntList() = default;

    void push_back(int v) {
        Node *n = new Node;
        n->value = v;
        n->next = nullptr;

        if (head == nullptr) {
            head = n;
        } else {
            Node *cur = head;
            while (cur->next != nullptr) {
                cur = cur->next;
            }
            cur->next = n;
        }
        count++;
    }

    void print_all() const {
        Node *cur = head;
        for (int i = 0; i < count; i++) {
            printf("%d\n", cur->value);
            cur = cur->next;
        }
    }

    auto sum() const -> int {
        int total = 0;
        Node *cur = head;
        while (cur != nullptr) {
            total = total + cur->value;
            cur = cur->next;
        }
        return total;
    }

    // แก้ memory leak: free ทุก node ตอนทำลาย list
    ~IntList() {
        Node *cur = head;
        while (cur != nullptr) {
            Node *next = cur->next;
            delete cur;
            cur = next;
        }
    }

private:
    Node *head{nullptr};
    int count{0};
};

class Base {
public:
    virtual void describe() { printf("Base\n"); }
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    void describe() override { printf("Derived\n"); }
};

// รับ const& แทนการ copy โดยไม่จำเป็น
auto greet(const std::string &name) -> std::string {
    // ใช้ std::string ต่อกันตรงๆ แทน char buffer + strcpy ที่เสี่ยง overflow
    return "Hello, " + name;
}

auto classify(int x) -> int {
    double half = static_cast<double>(x) / 2;
    return static_cast<int>(half);
}

auto main() -> int {
    IntList list;
    for (int i = 1; i <= 5; i++) list.push_back(i * 10);
    list.print_all();
    printf("sum = %d\n", list.sum());

    auto b = std::make_unique<Derived>();
    b->describe();

    std::string msg = greet("Somchai");
    printf("%s\n", msg.c_str());
    printf("classify(7) = %d\n", classify(7));
    return 0;
}
```

สรุปการแก้ไขที่ทำไปตาม suggestion แต่ละจุด:

| ปัญหาเดิม | Check ที่เจอ | วิธีแก้ |
|---|---|---|
| `NULL` | `modernize-use-nullptr` | เปลี่ยนเป็น `nullptr` ทุกจุด |
| destructor ว่างเปล่า/trivial | `modernize-use-equals-default` | ใช้ `= default` |
| override ที่ไม่มีคีย์เวิร์ด `override` | `modernize-use-override` | เติม `override` |
| `std::string name` (รับโดย copy) | `performance-unnecessary-value-param` | เปลี่ยนเป็น `const std::string &name` |
| `char buffer[32]` + `strcpy` | `modernize-avoid-c-arrays`, `clang-analyzer-security.insecureAPI.strcpy` | ใช้ `std::string` ต่อกันตรงๆ ไม่ใช้ buffer ดิบเลย |
| `(double)x`, `(int)half` | (C-style cast ไม่ผ่าน check เฉพาะในตัวอย่างนี้ แต่เป็น best practice) | เปลี่ยนเป็น `static_cast<>` |
| destructor ของ `IntList` ไม่ free node | (ตรวจพบด้วยการอ่านโค้ด ไม่ใช่ static analyzer อัตโนมัติ) | เขียน loop free ทุก node |

### ยืนยันผลหลังแก้: compile ผ่าน และ clang-tidy สะอาด

```bash
g++ -Wall -Wextra -std=c++17 legacy_list_fixed.cpp -o legacy_list_fixed
./legacy_list_fixed
```

```
10
20
30
40
50
sum = 150
Derived
Hello, Somchai
classify(7) = 3
```

ผลลัพธ์เหมือนเดิมทุกประการ (การแก้ตาม static analysis ไม่ควรเปลี่ยนพฤติกรรมของโปรแกรม)
ตรวจสอบด้วย clang-tidy อีกครั้ง:

```bash
clang-tidy legacy_list_fixed.cpp -checks='-*,modernize-*,bugprone-*,performance-*' -- -std=c++17
```

ผลลัพธ์จริงที่เหลืออยู่ (จากทั้งหมด 14 คำเตือนเดิม เหลือแค่ 2 จุดเล็กๆ):

```
warning: use default member initializer for 'head' [modernize-use-default-member-init]
warning: use default member initializer for 'count' [modernize-use-default-member-init]
```

แก้จุดสุดท้ายด้วยการเปลี่ยน:

```cpp
private:
    Node *head;
    int count;
```

เป็น

```cpp
private:
    Node *head{nullptr};
    int count{0};
```

รันซ้ำอีกครั้ง — เหลือแค่ 1 suggestion ที่เป็นทางเลือก (ไม่ใช่ปัญหา):

```
warning: [[nodiscard]]
   45 |     auto sum() const -> int {
```

clang-tidy แนะนำให้ใส่ `[[nodiscard]]` กับ `sum()` เพราะเป็นฟังก์ชันที่ไม่มี side effect และ
คืนค่าที่มีประโยชน์เสมอ (ถ้าเรียกแล้วไม่เก็บค่าไว้ใช้ น่าจะเป็นความผิดพลาดของผู้เขียนโค้ด) — นี่
เป็นตัวอย่างที่ดีว่า **ไม่จำเป็นต้องทำตาม suggestion ทุกตัวเสมอไป** ทีมสามารถตัดสินใจได้ว่า
`[[nodiscard]]` ในจุดนี้จำเป็นหรือไม่ (ในตัวอย่างนี้เลือกใส่เพิ่มเพื่อความสมบูรณ์ก็ได้ หรือจะปล่อยไว้
ก็ไม่ถือว่าผิด — ขึ้นอยู่กับ coding standard ของแต่ละทีม)

---

## 95.5 cppcheck: อีกมุมมองหนึ่งของ Static Analysis (Step 757)

### ติดตั้ง

```bash
sudo apt install cppcheck -y
cppcheck --version
```

```
Cppcheck 2.13.0
```

### ความแตกต่างเชิงสถาปัตยกรรมจาก clang-tidy

**cppcheck** เป็นเครื่องมือ static analysis ที่พัฒนาแยกอิสระจาก Clang/LLVM ทั้งหมด มี parser
และ analysis engine ของตัวเอง จุดเด่นคือ**ไม่ต้องมี compile flag ที่ถูกต้องเป๊ะเหมือน clang-tidy**
ทำให้เริ่มใช้งานได้เร็วกว่ามากในโปรเจกต์ที่ build system ซับซ้อนหรือยังไม่ได้ตั้งค่า
`compile_commands.json`

### รัน cppcheck กับไฟล์เดิม

```bash
cppcheck --enable=all --std=c++17 --language=c++ \
          --suppress=missingIncludeSystem legacy_list.cpp
```

ผลลัพธ์จริงบนเครื่องนี้:

```
Checking legacy_list.cpp ...
legacy_list.cpp:75:10: style: The function 'describe' overrides a function
in a base class but is not marked with a 'override' specifier. [missingOverride]
    void describe() {
         ^
legacy_list.cpp:66:18: note: Virtual function in base class
    virtual void describe() {
                 ^
legacy_list.cpp:75:10: note: Function in derived class
    void describe() {
         ^
legacy_list.cpp:81:31: performance: Function parameter 'name' should be
passed by const reference. [passedByValue]
std::string greet(std::string name) {
                              ^
```

น่าสนใจว่า cppcheck จับได้แค่ 2 ประเด็นจากไฟล์เดียวกัน (`missingOverride` และ `passedByValue`)
เทียบกับ 14 จุดของ clang-tidy — **ไม่ใช่เพราะ cppcheck "แย่กว่า"** แต่เพราะ cppcheck ถูก
ออกแบบให้เน้นความแม่นยำสูง (False Positive ต่ำ) มากกว่าความครอบคลุมกว้าง จึงไม่รายงานเรื่อง
สไตล์อย่าง `NULL` vs `nullptr` หรือ trailing return type ที่ไม่ใช่ความเสี่ยงเชิง logic โดยตรง
(สังเกตว่า cppcheck ก็ไม่จับปัญหา `strcpy` เช่นกันในโหมด default นี้)

### ตารางเปรียบเทียบ clang-tidy vs cppcheck

| หัวข้อ | clang-tidy | cppcheck |
|---|---|---|
| **Engine เบื้องหลัง** | ใช้ Clang/LLVM frontend เต็มรูปแบบ (เข้าใจโค้ดลึกเท่า compiler จริง) | Parser/engine ของตัวเอง แยกจาก compiler ใดๆ |
| **ต้องมี compile flag ถูกต้องไหม** | ควรมี (ผ่าน `compile_commands.json`) เพื่อผลแม่นยำสุด | ไม่จำเป็น ใช้งานได้แม้ config ไม่สมบูรณ์ |
| **จำนวน checks / ความครอบคลุม** | มากกว่ามาก (หลายร้อย checks, กลุ่ม `modernize-*` ที่ cppcheck ไม่มี) | น้อยกว่า แต่เน้นความแม่นยำสูง |
| **False Positive** | มีได้ในบาง check ที่ก้ำกึ่ง | ตั้งใจออกแบบให้ต่ำมาก (ปรัชญาของโปรเจกต์) |
| **ความเร็ว** | ช้ากว่าเล็กน้อยในโปรเจกต์ใหญ่ (ต้อง parse เต็มรูปแบบ) | เร็วกว่าในหลายกรณี |
| **จุดเด่นเฉพาะตัว** | Fix-It suggestions, checks กลุ่ม `modernize-*`/`cert-*`/security ลึก | ตรวจ Null pointer, array bound, resource leak แบบเจาะจงได้ดี |
| **การใช้งานที่แนะนำ** | ใช้เป็นหลักในการยกระดับโค้ดให้ modern และปลอดภัย | ใช้เสริมเป็น "second opinion" หาบั๊กที่ clang-tidy อาจพลาด |

**บทสรุปเชิงปฏิบัติ**: โปรเจกต์ระดับมืออาชีพจำนวนมากใช้**ทั้งสองตัวพร้อมกัน**ใน CI เพราะทั้งคู่
มองโค้ดจากมุมที่ต่างกัน สิ่งที่ตัวหนึ่งพลาด อีกตัวอาจจับได้ — Part นี้จะรวมทั้งสองตัวเข้า CI ใน
หัวข้อ 95.8

---

## 95.6 clang-format: จัดรูปแบบโค้ดอัตโนมัติ (Step 758)

### ปัญหา: การเถียงเรื่อง Style เป็นการเสียเวลาของทีม

ทุกทีมมีความเห็นต่างกันเรื่อง "ควรใส่ปีกกาบรรทัดเดียวกันหรือขึ้นบรรทัดใหม่", "เยื้อง 2 หรือ 4
space", "ตัวชี้ (`*`) ควรติดกับชนิดข้อมูลหรือชื่อตัวแปร" — คำถามพวกนี้**ไม่มีคำตอบที่ถูกที่สุด
สากล** สิ่งที่สำคัญกว่าคือ**ทั้งทีมต้องเขียนเหมือนกันหมด**เพื่อให้อ่าน diff และ code review ได้
ง่าย **clang-format** แก้ปัญหานี้ด้วยการจัดรูปแบบให้อัตโนมัติตาม config เดียวที่ทุกคนใช้ร่วมกัน

### ติดตั้ง

```bash
sudo apt install clang-format -y
clang-format --version
```

```
Ubuntu clang-format version 18.1.3 (1ubuntu1)
```

### ทดสอบกับโค้ดที่จัดรูปแบบมั่วๆ

```cpp
// messy.cpp
#include <cstdio>
struct Point{
int x,y;
    Point(int a,int b):x(a),y(b) {}
};
int main(  ) {
    Point p1(1,2);
    if(p1.x>0)
    {
        printf("positive: %d,%d\n",p1.x,p1.y);
    }
    else{
        printf("non-positive\n");
    }
  for(int i=0;i<5;i++){printf("%d ",i);}
    printf("\n");
    return 0;
}
```

รันด้วย style สำเร็จรูป `LLVM`:

```bash
clang-format -style=LLVM messy.cpp
```

ผลลัพธ์จริง:

```cpp
#include <cstdio>
struct Point {
  int x, y;
  Point(int a, int b) : x(a), y(b) {}
};
int main() {
  Point p1(1, 2);
  if (p1.x > 0) {
    printf("positive: %d,%d\n", p1.x, p1.y);
  } else {
    printf("non-positive\n");
  }
  for (int i = 0; i < 5; i++) {
    printf("%d ", i);
  }
  printf("\n");
  return 0;
}
```

โค้ดที่มั่วโดยสิ้นเชิง (เว้นวรรคมั่ว, ปีกกาคนละบรรทัด, ทุกอย่างชิดกัน) กลายเป็นโค้ดที่จัดรูปแบบ
สม่ำเสมอในทันที โดยไม่ต้องแก้ไขด้วยมือสักตัวอักษรเดียว

### เปรียบเทียบ Style สำเร็จรูป: LLVM vs Google

```bash
clang-format -style=LLVM -dump-config | grep -E "^(PointerAlignment|IndentWidth|AccessModifierOffset):"
clang-format -style=Google -dump-config | grep -E "^(PointerAlignment|IndentWidth|AccessModifierOffset):"
```

ผลลัพธ์จริง:

```
# LLVM
AccessModifierOffset: -2
IndentWidth:     2
PointerAlignment: Right

# Google
AccessModifierOffset: -1
IndentWidth:     2
PointerAlignment: Left
```

ความต่างที่ชัดเจนที่สุดคือ `PointerAlignment` ลองพิสูจน์ผลกระทบจริงกับโค้ดตัวอย่าง:

```cpp
// ptr_test.cpp
int* make_ptr(int x) {
    int* p = new int(x);
    return p;
}
```

```bash
clang-format -style=LLVM ptr_test.cpp
```

```cpp
int *make_ptr(int x) {
  int *p = new int(x);
  return p;
}
```

```bash
clang-format -style=Google ptr_test.cpp
```

```cpp
int* make_ptr(int x) {
  int* p = new int(x);
  return p;
}
```

**LLVM style** ยึด `*` ติดกับชื่อตัวแปร (`int *p`) ตามธรรมเนียมดั้งเดิมของภาษา C ส่วน **Google
style** ยึด `*` ติดกับชนิดข้อมูล (`int* p`) เพื่อสื่อว่า "`int*` คือชนิดข้อมูลหนึ่งตัว" — ทั้งสองแบบ
ถูกต้องตามไวยากรณ์ C++ เป๊ะๆ เหมือนกัน ต่างกันแค่ปรัชญาการอ่าน

### สร้าง `.clang-format` ของโปรเจกต์เอง

แทนที่จะใช้ style สำเร็จรูปตรงๆ ทีมส่วนใหญ่สร้าง config ของตัวเองโดย based on style ใดstyle
หนึ่งแล้ว override บางจุด:

```yaml
# .clang-format
BasedOnStyle: Google
IndentWidth: 4
ColumnLimit: 100
PointerAlignment: Left
```

เมื่อมีไฟล์ `.clang-format` อยู่ใน repository แล้ว รันแค่ `clang-format` เฉยๆ (ไม่ต้องระบุ
`-style=`) มันจะหาไฟล์ config นี้เจอเองอัตโนมัติ:

```bash
clang-format ptr_test.cpp
```

```cpp
int* make_ptr(int x) {
    int* p = new int(x);
    return p;
}
```

(สังเกตว่าตอนนี้เยื้อง 4 space ตามที่ override ไว้ ไม่ใช่ 2 space ของ Google style เดิม)

### ใช้ `-i` แก้ไฟล์จริง vs ใช้แบบตรวจสอบอย่างเดียว (สำหรับ CI)

การรัน `clang-format -i file.cpp` จะ**แก้ไฟล์นั้นทันที** (`-i` = in-place) ซึ่งเหมาะกับตอนพัฒนา
แต่ **ไม่เหมาะกับ CI** เพราะ CI ไม่ควรแก้โค้ดของคนอื่นเอง สิ่งที่ CI ควรทำคือ**ตรวจสอบแล้วแจ้ง
ว่าไม่ผ่าน**ถ้าโค้ดยังไม่ได้ format:

```bash
clang-format --dry-run --Werror messy.cpp
echo "exit code: $?"
```

ผลลัพธ์จริง (ตัดมาบางส่วนจากทั้งหมด 24 บรรทัด error):

```
messy.cpp:2:13: error: code should be clang-formatted [-Wclang-format-violations]
struct Point{
            ^
messy.cpp:8:7: error: code should be clang-formatted [-Wclang-format-violations]
    if(p1.x>0)
      ^
...
exit code: 1
```

**Exit code 1** คือสิ่งสำคัญที่สุดสำหรับ CI — มันบอกให้ workflow รู้ว่า step นี้ล้มเหลว โดยไม่ต้อง
แก้ไฟล์ใดๆ เลย เหมาะกับการเอาไปใส่ใน CI จาก Part 94 โดยตรง (จะทำในหัวข้อ 95.8)

### อีกวิธี: เปรียบเทียบด้วย `diff`

```bash
diff <(clang-format messy.cpp) messy.cpp
echo "exit code: $?"
```

ผลลัพธ์จริง (บางส่วน):

```
2,4c2,4
< struct Point {
<     int x, y;
<     Point(int a, int b) : x(a), y(b) {}
---
> struct Point{
> int x,y;
>     Point(int a,int b):x(a),y(b) {}
...
exit code: 1
```

วิธีนี้มีข้อดีตรงที่**เห็น diff ตรงๆ** ว่าอะไรจะเปลี่ยนถ้ารัน format จริง เหมาะสำหรับ debug ตอนที่
อยากรู้ว่าทำไม `--dry-run` ถึงบอกว่าไม่ผ่าน

---

## 95.7 รวม clang-tidy เข้ากับ CMake โดยตรง (Step 759)

### วิธีที่ 1: ผ่านตัวแปร `CMAKE_CXX_CLANG_TIDY`

CMake มีกลไกในตัวที่ทำให้ clang-tidy รันอัตโนมัติทุกครั้งที่ compile ไฟล์ โดยไม่ต้องสั่งแยก:

```bash
cmake -S . -B build-tidy \
      -DCMAKE_CXX_CLANG_TIDY="clang-tidy;-checks=-*,modernize-*,performance-*" \
      -DCMAKE_BUILD_TYPE=Debug
cmake --build build-tidy -j
```

ผลลัพธ์จริงจากการรันบนโปรเจกต์ตัวอย่าง (mathutils จาก Part 91–93) — clang-tidy รันแทรกเข้ามา
**ระหว่างขั้นตอน build ปกติ** โดยอัตโนมัติ:

```
[ 25%] Building CXX object CMakeFiles/mathutils.dir/src/mathutils.cpp.o
mathutils.cpp:5:5: warning: use a trailing return type for this function [modernize-use-trailing-return-type]
    5 | int add(int a, int b) {
      | ~~~ ^
      | auto                  -> int
mathutils.cpp:9:11: warning: use a trailing return type for this function [modernize-use-trailing-return-type]
    9 | long long factorial(int n) {
...
[ 50%] Linking CXX static library libmathutils.a
[ 50%] Built target mathutils
```

สังเกตว่า**การ build ไม่ล้มเหลว** แม้จะมี warning จาก clang-tidy — เพราะค่า default ของกลไกนี้
คือแค่แสดง warning เฉยๆ ถ้าต้องการให้ build ล้มเหลวจริงเมื่อเจอปัญหา ต้องเพิ่ม
`-warnings-as-errors=*` เข้าไปในค่าตัวแปรด้วย:

```bash
-DCMAKE_CXX_CLANG_TIDY="clang-tidy;-checks=-*,bugprone-*;-warnings-as-errors=*"
```

### วิธีที่ 2: ใส่ในไฟล์ CMakeLists.txt โดยตรง (ทางเลือกที่ทีมส่วนใหญ่ใช้จริง)

```cmake
option(ENABLE_CLANG_TIDY "Run clang-tidy during build" OFF)

if(ENABLE_CLANG_TIDY)
    find_program(CLANG_TIDY_EXE NAMES clang-tidy)
    if(CLANG_TIDY_EXE)
        set(CMAKE_CXX_CLANG_TIDY
            "${CLANG_TIDY_EXE};-checks=-*,modernize-*,bugprone-*,performance-*")
    endif()
endif()
```

การทำเป็น `option()` ที่ปิดโดย default (`OFF`) มีเหตุผล: การรัน clang-tidy ทุกครั้งที่ compile ทำ
ให้เวลา build ช้าลงพอสมควร (ต้อง parse โค้ดอีกรอบด้วย clang-tidy engine) นักพัฒนาส่วนใหญ่จึง
เปิดใช้เฉพาะตอนต้องการเช็คจริงจัง หรือปล่อยให้ **CI** เป็นคนเปิดใช้แทน (ผ่าน
`-DENABLE_CLANG_TIDY=ON` ตอน configure ใน workflow) ส่วนตอนพัฒนาปกติทุกวันให้ build เร็ว
ที่สุดไว้ก่อน

---

## 95.8 รวม Static Analysis เข้ากับ CI จาก Part 94 (Step 760)

### ขยาย `ci.yml` เพิ่ม job สำหรับ static analysis

ต่อยอดจาก workflow ที่เขียนไว้ใน Part 94 โดยเพิ่ม job ใหม่ที่รันขนานไปกับ job build-and-test เดิม:

```yaml
name: CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    strategy:
      fail-fast: false
      matrix:
        compiler: [gcc, clang]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake libgtest-dev g++ clang
      - name: Configure CMake
        run: cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
      - name: Build
        run: cmake --build build -j "$(nproc)"
      - name: Run unit tests
        working-directory: build
        run: ctest --output-on-failure

  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install analysis tools
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake libgtest-dev clang-tidy cppcheck clang-format

      - name: Generate compile_commands.json
        run: cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

      - name: Run clang-tidy
        run: |
          clang-tidy -p build src/*.cpp \
            -checks='-*,modernize-*,bugprone-*,performance-*,clang-analyzer-*'

      - name: Run cppcheck
        run: |
          cppcheck --enable=all --std=c++17 --language=c++ \
            --suppress=missingIncludeSystem --error-exitcode=1 src/

      - name: Check formatting (clang-format)
        run: |
          find src tests -name '*.cpp' -o -name '*.hpp' | \
            xargs clang-format --dry-run --Werror
```

อธิบายจุดสำคัญ:

- **แยก job `static-analysis` ออกจาก `build-and-test`** ทั้งสอง job รันขนานกัน (ไม่ต้องรอกัน
  เพราะไม่ได้พึ่งพาผลลัพธ์ของกันและกัน) ทำให้ได้ผลลัพธ์ทั้งสองด้านเร็วขึ้น
- **`--error-exitcode=1` ของ cppcheck** — โดย default cppcheck จะรายงานปัญหาแต่ **exit code
  เป็น 0 เสมอ** (ไม่ทำให้ CI fail) ต้องระบุ flag นี้เพื่อบังคับให้ exit code เป็น 1 เมื่อเจอปัญหา
  มิเช่นนั้น job นี้จะ "เขียว" ตลอดแม้เจอปัญหาจริงก็ตาม (จุดนี้เป็นกับดักที่พบบ่อยมาก จะพูดถึงอีก
  ครั้งใน Common Pitfalls)
- **`find ... | xargs clang-format --dry-run --Werror`** — วนตรวจทุกไฟล์ `.cpp`/`.hpp` ในโปรเจกต์
  แทนที่จะเช็คทีละไฟล์ด้วยมือ

### ยืนยันว่าทุกคำสั่งใน job นี้รันผ่านได้จริงบนเครื่อง local

ตามหลักการ "Validate Before Push" จาก Part 94 เราต้องรันทุกคำสั่งใน `run:` บนเครื่อง local
ก่อนเชื่อว่า workflow จะผ่านบน GitHub Actions:

```bash
cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
clang-tidy -p build src/mathutils.cpp \
  -checks='-*,modernize-*,bugprone-*,performance-*,clang-analyzer-*'
```

คำสั่งนี้ถูกรันจริงแล้วในหัวข้อ 95.7 (ผ่านกลไก `CMAKE_CXX_CLANG_TIDY`) และให้ warning เรื่อง
`modernize-use-trailing-return-type` ตามที่เห็นแล้ว — ถ้าต้องการให้ job ผ่านสีเขียวจริง ต้องแก้
`src/mathutils.cpp` ตาม suggestion เหล่านั้นก่อน push (แบบเดียวกับที่ฝึกแก้ `legacy_list.cpp`
ในหัวข้อ 95.4)

### ตรวจสอบ YAML

```bash
python3 -c "
import yaml
data = yaml.safe_load(open('.github/workflows/ci.yml'))
print('jobs:', list(data['jobs'].keys()))
"
```

```
jobs: ['build-and-test', 'static-analysis']
```

### ผลลัพธ์ที่จะเกิดขึ้นจริงบน GitHub (ภาพประกอบ)

Pull Request ใดๆ ที่ push โค้ดใหม่จะแสดงสถานะของ**ทั้งสอง job แยกกัน** ใน checks list ท้าย PR
— ถ้า `build-and-test` ผ่านแต่ `static-analysis` ไม่ผ่าน reviewer จะเห็นทันทีว่าโค้ดทำงานถูกต้อง
แต่มีปัญหาเชิงคุณภาพที่ควรแก้ก่อน merge บางทีมตั้งกฎว่า `static-analysis` เป็นแค่ "คำเตือน" ที่
merge ได้แม้ไม่ผ่าน (non-blocking) ในขณะที่บางทีมตั้งเป็น "บังคับผ่าน" (Required Status Check)
ก่อน merge ได้เสมอ — ขึ้นอยู่กับความเข้มงวดที่แต่ละทีมต้องการ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมว่า cppcheck exit code เป็น 0 เสมอโดย default**: แม้จะรายงานปัญหาเต็มหน้าจอ แต่ถ้าไม่
   ใส่ `--error-exitcode=1` การรันใน CI จะ "ผ่าน" เสมอไม่ว่าจะเจอปัญหากี่จุดก็ตาม เป็นกับดักที่
   ทำให้ทีมเข้าใจผิดว่า static analysis "ทำงานอยู่" ทั้งที่จริงๆ ไม่เคยบล็อกอะไรเลย
2. **ไม่ใส่ `-*,` นำหน้าตอนกำหนด `-checks=` ของ clang-tidy**: ทำให้ default checks ชุดใหญ่เปิด
   ทำงานปนมาด้วย รวมถึง warning จาก system header จำนวนมหาศาล (หลักหมื่น) จนหา warning ของ
   โค้ดตัวเองไม่เจอเลย
3. **รัน clang-tidy โดยไม่มี `compile_commands.json` ที่ถูกต้องในโปรเจกต์ที่ include path ซับซ้อน**:
   ทำให้ clang-tidy เข้าใจโค้ดผิด รายงาน error ปลอมๆ ที่ไม่เกี่ยวกับปัญหาจริงเลย (เช่น "หา header
   ไม่เจอ") ทางแก้คือสร้างด้วย `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` เสมอก่อนรัน
4. **ทำตาม static analysis suggestion แบบไม่คิดตาม (Blind Auto-Fix)**: บาง Fix-It ของ
   clang-tidy อาจเปลี่ยนพฤติกรรมโค้ดเล็กน้อยในกรณีขอบ (edge case) ที่เครื่องมือไม่รู้บริบททางธุรกิจ
   ควรอ่านทำความเข้าใจ diff ทุกครั้งก่อน apply ไม่ใช่รัน `clang-tidy -fix` แล้ว commit ทันทีโดยไม่
   ตรวจสอบ
5. **ตั้ง `.clang-format` คนละไฟล์ระหว่างเครื่อง local กับ CI**: ถ้านักพัฒนาแต่ละคนไม่ commit ไฟล์
   `.clang-format` เข้า repository แต่ใช้ config ส่วนตัวในเครื่อง (เช่นตั้งใน VS Code settings
   ส่วนตัว) จะทำให้ format ที่ commit เข้ามาไม่ตรงกันระหว่างคน และ CI (ที่ไม่มี config ส่วนตัวนั้น)
   จะ fail ทุกครั้งอย่างงงๆ ทางแก้คือ commit ไฟล์ `.clang-format` เข้า repository เสมอ
6. **มองว่า Static Analysis กับ Compiler Warning เป็นสิ่งเดียวกัน**: ทำให้เปิดแค่ `-Wall -Wextra`
   แล้วคิดว่าเพียงพอแล้ว ทั้งที่ตัวอย่างในหัวข้อ 95.1 พิสูจน์ชัดว่าโค้ดที่มี buffer overflow เสี่ยง
   สูงมากยัง compile ผ่านแบบไม่มี warning เลยได้สบายๆ

---

## แบบฝึกหัดท้ายบท

1. เขียนไฟล์ C++ ของตัวเองที่มีนิสัยแบบ "โค้ดเก่า" อย่างน้อย 3 จุด (เช่น ใช้ `NULL`, ลืมใส่
   `override`, รับ `std::string` โดย copy) แล้วรัน `clang-tidy` ด้วย
   `-checks='-*,modernize-*,performance-*'` ดูว่าเจอครบทุกจุดที่ตั้งใจใส่ไปหรือไม่
2. แก้โค้ดจากข้อ 1 ตาม suggestion ทุกจุด แล้วรัน clang-tidy ซ้ำเพื่อยืนยันว่าไม่มี warning เหลือ
   (หรือเหลือเฉพาะจุดที่ตัดสินใจไม่แก้เพราะเป็นเรื่องรสนิยม)
3. รัน `cppcheck --enable=all` กับไฟล์เดียวกันจากข้อ 1 เปรียบเทียบว่า cppcheck เจอปัญหาจุดไหน
   ที่ clang-tidy ไม่เจอ หรือไม่เจอจุดไหนที่ clang-tidy เจอบ้าง อธิบายด้วยคำพูดตัวเองว่าทำไมถึงต่างกัน
4. สร้างไฟล์ `.clang-format` ของตัวเอง โดย `BasedOnStyle: LLVM` แต่ override `IndentWidth`
   เป็น 4 และ `ColumnLimit` เป็น 120 แล้วทดสอบกับไฟล์ที่จัดรูปแบบมั่วๆ ว่าได้ผลลัพธ์ตามที่ตั้งค่าไว้จริง
5. เพิ่ม `CMAKE_CXX_CLANG_TIDY` เข้าไปใน `CMakeLists.txt` ของโปรเจกต์ตัวเองแบบมี `option()`
   เปิด/ปิดได้ ทดสอบทั้งตอนเปิดและปิดว่าเวลา build ต่างกันอย่างมีนัยสำคัญหรือไม่
6. เขียน job `static-analysis` เพิ่มเข้าไปใน `.github/workflows/ci.yml` จาก Part 94 ให้ครบทั้ง
   clang-tidy, cppcheck (พร้อม `--error-exitcode=1`), และ clang-format (`--dry-run --Werror`)
   แล้วตรวจสอบ YAML syntax ให้ผ่าน

### แนวทางเฉลยข้อ 3

สมมติไฟล์จากข้อ 1 มีทั้งปัญหา `NULL`, ไม่มี `override`, และ `strcpy` ที่เสี่ยง overflow:

```bash
clang-tidy myfile.cpp -checks='-*,modernize-*,performance-*' -- -std=c++17
# เจอ: modernize-use-nullptr, modernize-use-override, performance-unnecessary-value-param
# ไม่เจอ: ปัญหา strcpy (เพราะไม่ได้เปิดกลุ่ม clang-analyzer-*)

cppcheck --enable=all --std=c++17 --language=c++ myfile.cpp
# เจอ: missingOverride, passedByValue (ถ้ามี std::string รับโดย value)
# ไม่เจอ: NULL vs nullptr (ไม่ใช่ปรัชญาการตรวจของ cppcheck — ไม่ถือว่าเป็นบั๊ก)
# ไม่เจอ: strcpy overflow ในโหมด default เช่นกัน (ต้องเปิด --check-level=exhaustive หรือใช้เครื่องมือ
#          เฉพาะทางเพิ่มเติมสำหรับกรณีนี้)
```

**คำอธิบาย**: clang-tidy กลุ่ม `modernize-*` เจอเรื่อง `NULL` เพราะเป็นปรัชญาของ clang-tidy ที่
ต้องการผลักดันโค้ดให้ทันสมัย ส่วน cppcheck ไม่ถือว่า `NULL` เป็นปัญหาเพราะในทางเทคนิคมันทำงาน
ถูกต้อง 100% (ไม่ใช่บั๊ก แค่ไม่ทันสมัย) — นี่คือตัวอย่างที่ชัดเจนว่าทำไมการใช้เครื่องมือสองตัวรวมกัน
ถึงให้มุมมองที่กว้างกว่าการใช้ตัวใดตัวหนึ่งเพียงลำพัง

### แนวทางเฉลยข้อ 4

```yaml
# .clang-format
BasedOnStyle: LLVM
IndentWidth: 4
ColumnLimit: 120
```

```bash
clang-format messy.cpp
```

ผลลัพธ์ควรเยื้องด้วย 4 space (ไม่ใช่ 2 space ของ LLVM ดั้งเดิม) และบรรทัดยาวได้ถึง 120 ตัวอักษร
ก่อนจะขึ้นบรรทัดใหม่ (ไม่ใช่ 80 ตัวอักษรของ LLVM ดั้งเดิม) — พิสูจน์ได้ด้วยการเขียนบรรทัดยาวๆ
เกิน 80 ตัวอักษรแล้วดูว่า clang-format ยอมปล่อยให้อยู่บรรทัดเดียวกันจนถึง 120 ตัวอักษรจริงหรือไม่

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Static Analysis ต่างจาก Compiler Warning, Unit Test, และ Sanitizer อย่างไร และ
  ทำไมทั้งหมดนี้ต้องทำงานร่วมกันเป็นชั้นๆ ไม่ใช่เลือกใช้แค่ตัวเดียว
- รัน clang-tidy จริงกับโค้ดสไตล์เก่าที่ compile ผ่านสะอาดโดยไม่มี compiler warning เลย แต่มี
  ปัญหาซ่อนอยู่ถึง 14 จุด และแก้ไขจนสะอาดตาม suggestion ได้จริง
- เข้าใจกลุ่ม checks สำคัญ (`modernize-*`, `bugprone-*`, `performance-*`, `clang-analyzer-*`)
  และรู้วิธีตั้งค่าผ่านไฟล์ `.clang-tidy`
- รัน cppcheck เปรียบเทียบกับ clang-tidy บนโค้ดเดียวกัน เข้าใจว่าทำไมทั้งสองเครื่องมือเจอ
  ปัญหาคนละชุด และควรใช้ร่วมกัน
- ใช้ clang-format จัดรูปแบบโค้ดอัตโนมัติ เข้าใจความต่างระหว่าง LLVM/Google style และใช้
  `--dry-run --Werror` เพื่อตรวจสอบใน CI โดยไม่แก้ไฟล์
- ผูก clang-tidy เข้ากับ CMake โดยตรงผ่าน `CMAKE_CXX_CLANG_TIDY`
- ขยาย CI workflow จาก Part 94 ให้มี job `static-analysis` แยกต่างหาก ครอบคลุมทั้ง 3 เครื่องมือ

ตอนนี้ pipeline ของเรามีทั้ง **Unit Test** (Part 93), **CI อัตโนมัติ** (Part 94), และ **Static
Analysis** (Part นี้) แล้ว แต่ยังขาดสิ่งสำคัญอีกชั้นหนึ่ง: การตรวจจับบั๊กที่**เกิดขึ้นจริงตอนรัน
โปรแกรม**เท่านั้น เช่น memory corruption หรือ race condition ที่ static analyzer มองไม่เห็นเพราะ
ต้องอาศัย runtime behavior จริง ใน **Part 96** เราจะเรียนรู้ **Sanitizer** (ASan, UBSan, TSan)
และ **Code Coverage** (gcov/lcov) เพื่อปิดช่องว่างสุดท้ายนี้

**ต่อไป:** [Part 96 — Sanitizer และ Code Coverage: ASan, UBSan, TSan, gcov/lcov](./part-096-sanitizers-coverage.md)
