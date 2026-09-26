# Part 56: Function Template และ Class Template (Step 441–448)

> Module E — Templates, Generic Programming และ STL | Part 56 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 441–448
> Part ก่อนหน้า: [Part 55 — โปรเจกต์ Library Management System](./part-055-library-system-project.md) | Part ถัดไป: [Part 57 — Template Specialization และ Variadic Template](./part-057-template-specialization.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายปัญหาของการเขียนโค้ดซ้ำๆ สำหรับแต่ละชนิดข้อมูล และเข้าใจว่า Template แก้ปัญหานี้
   อย่างไรในระดับ compile-time (ไม่ใช่ runtime)
2. เขียน **Function Template** ด้วย syntax `template <typename T>` ได้อย่างถูกต้อง
3. เข้าใจกลไก **Template Argument Deduction** — compiler รู้ได้อย่างไรว่า `T` ควรเป็นชนิดใด
   จาก argument ที่ส่งเข้าไป และรู้ว่าเมื่อไหร่ต้องระบุ type เอง (explicit template argument)
4. เขียน **Class Template** ของตัวเอง (เช่น `Stack<T>`) ที่ทำงานได้กับชนิดข้อมูลใดก็ได้
5. อธิบายกระบวนการ **Template Instantiation** ที่เกิดขึ้นตอน compile-time ได้อย่างละเอียด
   พร้อมพิสูจน์ด้วยเครื่องมือจริง (`nm`) ว่า compiler สร้างโค้ดแยกสำหรับแต่ละชนิดข้อมูลจริงๆ
6. เขียน Template ที่มี **หลาย Template Parameter** พร้อมกัน (เช่น `Pair<T1, T2>`)
7. เขียนและใช้งาน **Non-Type Template Parameter** (เช่น ขนาดของ array ที่กำหนดตอน compile-time)
8. อธิบายได้ว่าทำไม Template ต้อง implement อยู่ใน Header File เสมอ และแก้ปัญหา
   **Linker Error** ที่เกิดจากการแยก implementation ไปไว้ใน `.cpp` แยกต่างหาก

---

## 56.1 ปัญหาที่ Template แก้: เขียนโค้ดซ้ำๆ สำหรับแต่ละ Type (Step 441)

ตลอด Module D เราเขียนฟังก์ชันและ class ที่ทำงานกับชนิดข้อมูล**เฉพาะเจาะจง**เสมอ เช่น
ฟังก์ชันหาค่ามากสุดระหว่างตัวเลขสองตัว:

```cpp
int maxInt(int a, int b) {
    return (a > b) ? a : b;
}
```

ปัญหาเกิดขึ้นทันทีที่เราต้องการหาค่ามากสุดของ `double` บ้าง `char` บ้าง เราต้อง**เขียนฟังก์ชันซ้ำ**
ที่มี logic เหมือนเดิมทุกตัวอักษร ต่างกันแค่ type:

```cpp
#include <iostream>

// ปัญหา: ต้องเขียนฟังก์ชัน max ซ้ำๆ สำหรับทุก type ที่ต้องการใช้งาน
int maxInt(int a, int b) {
    return (a > b) ? a : b;
}

double maxDouble(double a, double b) {
    return (a > b) ? a : b;
}

char maxChar(char a, char b) {
    return (a > b) ? a : b;
}

int main() {
    std::cout << "maxInt(3, 7) = " << maxInt(3, 7) << '\n';
    std::cout << "maxDouble(3.5, 2.1) = " << maxDouble(3.5, 2.1) << '\n';
    std::cout << "maxChar('a', 'z') = " << maxChar('a', 'z') << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 problem.cpp -o problem
./problem
```

```
maxInt(3, 7) = 7
maxDouble(3.5, 2.1) = 3.5
maxChar('a', 'z') = z
```

โค้ดทำงานถูกต้อง แต่มีปัญหาสำคัญ 3 ข้อ:

1. **ละเมิดหลักการ DRY (Don't Repeat Yourself)** — logic เดียวกันถูกเขียนซ้ำ 3 ครั้ง ถ้าพบบั๊ก
   ในอนาคต (เช่น อยากเปลี่ยนเงื่อนไขจาก `>` เป็น `>=`) ต้องแก้ทั้ง 3 ที่พร้อมกัน และเสี่ยงลืมแก้
   บางจุด
2. **ไม่ขยายได้ (ไม่ Scalable)** — ถ้าต้องการ `maxFloat`, `maxLong`, `maxString` เพิ่มขึ้นมา ต้อง
   เขียนฟังก์ชันใหม่ทุกครั้งที่มีชนิดข้อมูลใหม่เกิดขึ้น
3. **ตั้งชื่อยาก** — ต้องคิดชื่อฟังก์ชันที่ไม่ซ้ำกันสำหรับแต่ละ type (`maxInt`, `maxDouble`, ...)
   ทั้งที่ผู้ใช้ต้องการแค่ "หาค่ามากสุด" เท่านั้น ไม่ได้สนใจว่าต้องเรียกชื่อฟังก์ชันอะไร

เราอาจคิดว่า **Function Overloading** (Part 44) ช่วยได้ เพราะสามารถตั้งชื่อฟังก์ชันเดียวกันได้
(`max(int, int)`, `max(double, double)`, `max(char, char)`) แต่นั่นแก้ได้แค่ปัญหาข้อ 3 เท่านั้น
— **โค้ดยังคงต้องเขียนซ้ำทุกตัวอักษร** สำหรับแต่ละ overload อยู่ดี

นี่คือปัญหาที่ **Template** ถูกออกแบบมาเพื่อแก้ไขโดยเฉพาะ: เขียนโค้ด**เพียงครั้งเดียว** โดยใช้
"ชนิดข้อมูลที่ยังไม่ระบุ" เป็น placeholder แล้วปล่อยให้ **compiler** เป็นผู้สร้างฟังก์ชันจริงสำหรับ
แต่ละ type ที่ถูกใช้งานจริงให้เองโดยอัตโนมัติที่ **compile-time** — แนวคิดนี้เรียกว่า
**Generic Programming** และเป็นรากฐานสำคัญที่สุดของ **STL (Standard Template Library)**
ที่เราจะใช้งานอย่างเข้มข้นตลอด Module E นี้

---

## 56.2 Function Template: Syntax และการทำงาน (Step 442)

เราสามารถเขียนฟังก์ชัน `max` แบบเดียวที่ทำงานได้กับทุกชนิดข้อมูลได้ด้วย **Function Template**:

```cpp
#include <iostream>
#include <string>

// Function Template: T คือ "ตัวยึดตำแหน่ง (placeholder)" ของ type ที่จะกำหนดตอนเรียกใช้งาน
template <typename T>
T myMax(T a, T b) {
    return (a > b) ? a : b;
}

int main() {
    std::cout << "myMax(3, 7) = " << myMax(3, 7) << '\n';
    std::cout << "myMax(3.5, 2.1) = " << myMax(3.5, 2.1) << '\n';
    std::cout << "myMax('a', 'z') = " << myMax('a', 'z') << '\n';
    std::cout << "myMax string = " << myMax(std::string("apple"), std::string("banana")) << '\n';

    // เรียกแบบระบุ type ชัดเจน (explicit template argument)
    std::cout << "myMax<double>(3, 7) = " << myMax<double>(3, 7) << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 function_template.cpp -o function_template
./function_template
```

```
myMax(3, 7) = 7
myMax(3.5, 2.1) = 3.5
myMax('a', 'z') = z
myMax string = banana
myMax<double>(3, 7) = 7
```

### อธิบายทีละส่วน

- **`template <typename T>`**: บรรทัดนี้ประกาศว่า "ฟังก์ชันข้างล่างนี้เป็น Template ที่มี
  Template Parameter ชื่อ `T`" — คำว่า `typename` บอกว่า `T` คือ**ชนิดข้อมูล** ที่ยังไม่ระบุ
  (สามารถใช้คำว่า `class` แทน `typename` ได้ด้วย ทั้งสองคำมีความหมายเหมือนกันทุกประการในบริบทนี้
  — `typename` เป็นที่นิยมมากกว่าในโค้ดสมัยใหม่เพราะสื่อความหมายชัดกว่าว่าไม่จำเป็นต้องเป็น
  `class` เท่านั้น)
- **`T myMax(T a, T b)`**: นิยามฟังก์ชันตามปกติ แต่ใช้ `T` แทนชนิดข้อมูลที่แน่นอน ทั้ง
  return type และ parameter type
- **`T` ไม่ใช่ตัวแปร ไม่ใช่ macro** — มันคือ **placeholder ของชนิดข้อมูล** ที่ compiler จะแทนที่
  ด้วยชนิดข้อมูลจริงตอน compile-time เท่านั้น ไม่มีสิ่งใดเกี่ยวกับ `T` หลงเหลืออยู่ตอน runtime
  เลย (ต่างจากภาษาที่มี Generic แบบ runtime เช่น type erasure ใน Java รุ่นเก่า)
- **การเรียกใช้** `myMax(3, 7)` — เราไม่ต้องระบุ `<int>` เอง เพราะ compiler **อนุมาน
  (deduce)** ได้จาก argument ที่ส่งเข้าไปว่า `T` ควรเป็น `int` กระบวนการนี้เรียกว่า
  **Template Argument Deduction** ซึ่งจะอธิบายละเอียดใน 56.3
- **`myMax<double>(3, 7)`**: การระบุ `<double>` ชัดเจนหลังชื่อฟังก์ชันเรียกว่า
  **Explicit Template Argument** บังคับให้ `T = double` ทำให้ `3` และ `7` (ที่เป็น `int`
  โดยธรรมชาติ) ถูกแปลง (implicit conversion) เป็น `double` ก่อนเข้าฟังก์ชัน

### เปรียบเทียบกับ Macro

มือใหม่บางคนอาจสงสัยว่า Template ต่างจาก Macro (`#define`, Part 14) อย่างไร ในเมื่อทั้งคู่ดู
เหมือน "แทนที่ข้อความ" คำตอบคือ**ต่างกันโดยสิ้นเชิง**:

| ประเด็น | Macro (`#define`) | Template |
|---|---|---|
| ทำงานตอนไหน | Preprocessing (ก่อน compile) | Compilation (มี type checking) |
| Type Safety | ไม่มี — เป็นแค่ text substitution ตรงๆ | มี — compiler ตรวจสอบ type ให้เต็มรูปแบบ |
| Debug ได้ไหม | ยาก เพราะ debugger เห็นแค่โค้ดที่ขยายแล้ว | ได้ปกติ เหมือนฟังก์ชันทั่วไป |
| Scope | ไม่มี scope เป็น global เสมอ | มี scope ตามปกติของภาษา |
| ผลข้างเคียง | เสี่ยง bug จาก operator precedence (ทบทวน Part 14) | ไม่มีปัญหานี้เลย เพราะเป็นโค้ด C++ จริง |

**กฎทองของ Modern C++**: ถ้าต้องการเขียนโค้ดที่ทำงานกับหลาย type ให้ใช้ **Template** เสมอ
ไม่ใช้ Macro — Template ปลอดภัยกว่า ตรวจสอบ error ได้ตอน compile-time และ debug ได้ง่ายกว่ามาก

---

## 56.3 Template Argument Deduction (Step 443)

**Template Argument Deduction** คือกระบวนการที่ compiler วิเคราะห์ argument ที่เราส่งเข้าไป
แล้ว "เดา" ว่า Template Parameter ควรถูกแทนที่ด้วยชนิดข้อมูลอะไร โดยที่เราไม่ต้องระบุเอง

```cpp
#include <iostream>

template <typename T>
T myMax(T a, T b) {
    return (a > b) ? a : b;
}

// Template ที่มีหลาย type parameter — คนละ type กันได้
template <typename T, typename U>
void printPair(T first, U second) {
    std::cout << "(" << first << ", " << second << ")\n";
}

int main() {
    // Deduction สำเร็จ: ทั้งสอง argument เป็น int เหมือนกัน -> T = int
    std::cout << myMax(10, 20) << '\n';

    // ถ้า argument คนละ type กัน ต้องระบุ type เอง หรือแปลงให้ตรงกันก่อน
    // myMax(10, 3.5);              // จะ compile error: deduction ขัดแย้งกัน (T=int vs T=double)
    std::cout << myMax<double>(10, 3.5) << '\n';   // ระบุ T=double ชัดเจน -> 10 ถูกแปลงเป็น 10.0

    printPair(1, "hello");     // T=int, U=const char*
    printPair(3.14, 'x');      // T=double, U=char

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 deduction.cpp -o deduction
./deduction
```

```
20
10
(1, hello)
(3.14, x)
```

### สิ่งที่ต้องเข้าใจให้ชัดเจน

- **Deduction วิเคราะห์จาก argument ทุกตัวที่ตรงกับ Template Parameter ตัวเดียวกัน** — ใน
  `T myMax(T a, T b)` ทั้ง `a` และ `b` ผูกกับ `T` ตัวเดียวกัน ถ้าเราเรียก `myMax(10, 3.5)` โดยไม่
  ระบุ type เอง compiler จะเห็นว่า argument แรกบอกว่า `T` ควรเป็น `int` แต่ argument ที่สองบอกว่า
  `T` ควรเป็น `double` — เกิด**ความขัดแย้ง (ambiguity)** และจะ **compile error ทันที**
  (ข้อความ error จะประมาณ `no matching function for call` หรือ `template argument deduction/
  substitution failed`) นี่คือจุดที่มือใหม่งงบ่อยที่สุด
- **วิธีแก้เมื่อ type ไม่ตรงกัน**: มีสองทางเลือก
  1. ระบุ type เอง: `myMax<double>(10, 3.5)` — บังคับให้ argument ทุกตัวถูกแปลงเป็น type
     ที่ระบุก่อนเข้าฟังก์ชัน
  2. แปลง argument ให้ตรงชนิดกันเองก่อนเรียก: `myMax(10, 3.5)` → `myMax(10.0, 3.5)` หรือ
     `myMax(static_cast<double>(10), 3.5)`
- **Template หลาย parameter ไม่มีปัญหานี้** — `printPair(T first, U second)` มี `T` และ `U`
  แยกกันคนละตัว ดังนั้น `first` กับ `second` เป็นคนละ type กันได้อย่างอิสระ compiler deduce
  แยกกันสำหรับแต่ละ parameter
- **Deduction ทำงานเฉพาะตอนเรียกฟังก์ชันเท่านั้น** — ต่างจาก Class Template (จะเห็นใน 56.4)
  ที่ก่อน C++17 ต้องระบุ type ตอนสร้าง object เสมอ (ตั้งแต่ C++17 มี **Class Template Argument
  Deduction (CTAD)** ที่ deduce type ให้ตอนสร้าง object ได้เหมือนกัน จะพูดถึงใน Part 72)

---

## 56.4 Class Template: เขียน `Stack<T>` ของตัวเอง (Step 444)

Template ไม่ได้ใช้ได้แค่กับฟังก์ชันเท่านั้น เราสามารถเขียน **Class Template** ที่ทำงานกับชนิด
ข้อมูลใดก็ได้เช่นกัน ลองเขียน `Stack<T>` เวอร์ชันของตัวเอง (เทียบกับ Stack ที่เขียนด้วย C ล้วนๆ
ใน Part 20 ที่ต้องเขียนแยกสำหรับแต่ละชนิดข้อมูล หรือใช้ `void*` ที่เสี่ยง type-unsafe):

```cpp
#include <iostream>
#include <stdexcept>
#include <string>
#include <vector>

// ===== Class Template: Stack<T> เขียนเอง =====
template <typename T>
class Stack {
public:
    void push(const T& value) {
        data_.push_back(value);
    }

    void pop() {
        if (empty()) {
            throw std::out_of_range("Stack::pop: stack ว่างเปล่า ไม่สามารถ pop ได้");
        }
        data_.pop_back();
    }

    T& top() {
        if (empty()) {
            throw std::out_of_range("Stack::top: stack ว่างเปล่า ไม่มี top");
        }
        return data_.back();
    }

    const T& top() const {
        if (empty()) {
            throw std::out_of_range("Stack::top: stack ว่างเปล่า ไม่มี top");
        }
        return data_.back();
    }

    bool empty() const noexcept { return data_.empty(); }
    std::size_t size() const noexcept { return data_.size(); }

private:
    std::vector<T> data_;
};

int main() {
    Stack<int> intStack;
    intStack.push(1);
    intStack.push(2);
    intStack.push(3);
    std::cout << "top = " << intStack.top() << ", size = " << intStack.size() << '\n';
    intStack.pop();
    std::cout << "หลัง pop: top = " << intStack.top() << ", size = " << intStack.size() << '\n';

    Stack<std::string> stringStack;
    stringStack.push("แรก");
    stringStack.push("สอง");
    std::cout << "stringStack.top() = " << stringStack.top() << '\n';

    try {
        Stack<double> emptyStack;
        emptyStack.pop();
    } catch (const std::out_of_range& e) {
        std::cout << "จับ exception ได้ถูกต้อง: " << e.what() << '\n';
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 stack_template.cpp -o stack_template
./stack_template
```

```
top = 3, size = 3
หลัง pop: top = 2, size = 2
stringStack.top() = สอง
จับ exception ได้ถูกต้อง: Stack::pop: stack ว่างเปล่า ไม่สามารถ pop ได้
```

### อธิบายการออกแบบ

- **`template <typename T> class Stack { ... };`**: ประกาศ Template Parameter ให้ทั้ง class
  — `T` สามารถใช้ได้ทุกที่ภายใน class เสมือนเป็นชนิดข้อมูลจริง (ใน member function, return
  type, member variable)
- **`std::vector<T> data_;`**: เราไม่ได้เขียน dynamic array เองด้วยมือ (แบบที่ทำใน Part 11
  หรือ Part 20) แต่ใช้ `std::vector<T>` ของ STL เป็น "ตัวเก็บข้อมูลจริง" ข้างใน — นี่คือรูปแบบ
  ที่พบบ่อยมากในโค้ดจริง: เขียน class ห่อหุ้ม (wrapper) รอบ container ของ STL เพื่อจำกัด
  interface ให้ตรงกับพฤติกรรมที่ต้องการ (ในที่นี้คือพฤติกรรมแบบ LIFO ของ Stack เท่านั้น ไม่เปิด
  ให้เข้าถึง element กลาง array แบบสุ่มเหมือน `vector` ธรรมดา)
- **สังเกตว่า `Stack<int>` และ `Stack<std::string>` เป็นคนละ type กันโดยสิ้นเชิง** —
  ในสายตาของ compiler มันคือ class คนละตัวกันเลย (แค่ถูกสร้างจาก template เดียวกัน) ดังนั้น
  `Stack<int>` ไม่สามารถแปลงเป็น `Stack<std::string>` ได้โดยตรง เหมือนที่ `int` แปลงเป็น
  `std::string` ตรงๆ ไม่ได้เช่นกัน
- **`push`/`pop`/`top`/`empty`/`size`**: ออกแบบ interface ให้เหมือนกับ `std::stack` ของจริงใน
  STL (จะเรียนใน Part 58) โดยตั้งใจ — เพื่อให้ผู้เรียนคุ้นเคยกับชื่อ method มาตรฐานที่ STL ใช้
  ก่อนที่จะไปเรียนใช้งาน `std::stack` ตัวจริงในบทถัดๆ ไป
- **`T& top()` และ `const T& top() const`**: เขียน overload สองแบบ (ทบทวน const-correctness
  จาก Part 43 และ 47) เพื่อให้ทั้ง `Stack<T>` แบบ mutable และ `const Stack<T>` เรียก `top()` ได้
  ทั้งคู่

---

## 56.5 Template Instantiation: สิ่งที่เกิดขึ้นจริงตอน Compile-Time (Step 445)

คำว่า **Instantiation** หมายถึงกระบวนการที่ compiler "สร้าง" โค้ดจริง (ฟังก์ชันหรือ class จริง)
จาก Template โดยแทนที่ Template Parameter ด้วยชนิดข้อมูลที่ถูกใช้งานจริงในโปรแกรม —
**Template เองไม่ใช่โค้ดที่รันได้** มันเป็นเพียง "พิมพ์เขียว (blueprint)" ที่ compiler ใช้สร้าง
โค้ดจริงตอน compile-time เท่านั้น

ลองพิสูจน์ด้วยตาตัวเอง:

```cpp
#include <iostream>

template <typename T>
T myMax(T a, T b) {
    return (a > b) ? a : b;
}

int main() {
    // แต่ละบรรทัดนี้บังคับให้ compiler สร้าง (instantiate) ฟังก์ชันคนละตัวจาก template เดียวกัน
    std::cout << myMax(1, 2) << '\n';          // instantiate myMax<int>
    std::cout << myMax(1.5, 2.5) << '\n';      // instantiate myMax<double>
    std::cout << myMax('a', 'b') << '\n';      // instantiate myMax<char>
    return 0;
}
```

คอมไพล์เป็น object file แล้วใช้ `nm` (เครื่องมือดู symbol table ที่เคยเห็นบ้างแล้วใน Part 39)
ตรวจสอบว่า compiler สร้างฟังก์ชันอะไรไว้จริงบ้าง:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -c instantiation.cpp -o instantiation.o
nm -C instantiation.o | grep myMax
```

```
0000000000000000 W char myMax<char>(char, char)
0000000000000000 W double myMax<double>(double, double)
0000000000000000 W int myMax<int>(int, int)
```

**นี่คือหลักฐานที่ชัดเจนที่สุด**: แม้เราจะเขียน `myMax` แค่ครั้งเดียวในซอร์สโค้ด แต่ compiler
สร้างฟังก์ชันจริงถึง **3 ตัวแยกกัน** — `myMax<int>`, `myMax<double>`, และ `myMax<char>` — ลงใน
object file จริงๆ (สังเกตตัว `-C` ที่ส่งให้ `nm` เพื่อ demangle ชื่อ C++ ที่ปกติจะถูกเข้ารหัส
เป็นชื่อประหลาดๆ ให้กลับมาอ่านง่าย)

### เหตุผลเบื้องหลัง (`W` คืออะไร)

ตัวอักษร `W` หน้าแต่ละบรรทัดหมายถึง **Weak Symbol** — เพราะฟังก์ชันที่ถูก instantiate จาก
template อาจถูกสร้างซ้ำในหลาย object file ถ้า header ที่มี template นั้นถูก `#include` ในหลาย
`.cpp` (แต่ละ Translation Unit สร้างสำเนาของตัวเองตอน compile) linker จึงต้องรู้ว่าเจอ symbol
ซ้ำแบบนี้ **ไม่ใช่ error** ให้เลือกสำเนาไหนสำเนาหนึ่งมาใช้แล้วทิ้งสำเนาที่เหลือ — กลไกนี้เป็นสิ่ง
ที่ทำให้การมี template implementation ซ้ำกันในหลายไฟล์ทำงานได้อย่างถูกต้อง (จะอธิบายเพิ่มเติมใน
56.8 ว่าทำไมเรื่องนี้ถึงสำคัญมาก)

### ผลกระทบเชิงปฏิบัติ: ขนาดโปรแกรมและเวลา Compile

การที่ compiler ต้องสร้างโค้ดแยกสำหรับทุก type ที่ใช้งานจริง มีผลกระทบสองด้าน:

1. **ขนาดไฟล์ executable ใหญ่ขึ้น (Code Bloat)** — ถ้าโปรแกรมใช้ `myMax` กับ 10 ชนิดข้อมูลที่
   ต่างกัน จะมีโค้ด assembly แยกกัน 10 ชุดฝังอยู่ในไฟล์สุดท้าย (ต่างจากฟังก์ชันธรรมดาที่มีโค้ด
   ชุดเดียว) ปัญหานี้จะกล่าวถึงเชิงลึกอีกครั้งเมื่อพูดถึง **Template Metaprogramming**
   ใน Part 78
2. **เวลา Compile นานขึ้น** — ยิ่งใช้ template กับหลาย type มาก compiler ยิ่งต้องทำงานสร้างโค้ด
   มากตามไปด้วย โปรเจกต์ใหญ่ที่ใช้ template หนักๆ (เช่นโปรเจกต์ที่ใช้ STL/Boost อย่างเข้มข้น)
   จึงมักใช้เวลา compile นานกว่าโค้ด C ธรรมดาอย่างมีนัยสำคัญ

สิ่งเหล่านี้คือ **trade-off** ที่ต้องแลกกับความสามารถในการเขียนโค้ด generic ที่ type-safe และ
ไม่มี overhead ตอน runtime เลย (ต่างจากภาษาที่ใช้ Generic ผ่าน runtime polymorphism ซึ่งมี
ค่าใช้จ่ายด้าน performance ตอนรันจริง) — โดยรวมแล้วถือเป็นข้อแลกเปลี่ยนที่คุ้มค่ามากในภาษา C++

---

## 56.6 Template ที่มีหลาย Template Parameter (Step 446)

Class Template สามารถมี Template Parameter ได้มากกว่าหนึ่งตัว เหมาะสำหรับข้อมูลที่ต้องเก็บ
คู่ค่าที่เป็นคนละชนิดกัน:

```cpp
#include <iostream>
#include <string>

// Class Template ที่มีหลาย template parameter: T1 และ T2 เป็นคนละ type กันได้
template <typename T1, typename T2>
class Pair {
public:
    Pair(T1 first, T2 second) : first_(std::move(first)), second_(std::move(second)) {}

    const T1& first() const noexcept { return first_; }
    const T2& second() const noexcept { return second_; }

    void print(std::ostream& os) const {
        os << "(" << first_ << ", " << second_ << ")";
    }

private:
    T1 first_;
    T2 second_;
};

template <typename T1, typename T2>
std::ostream& operator<<(std::ostream& os, const Pair<T1, T2>& p) {
    p.print(os);
    return os;
}

int main() {
    Pair<std::string, int> studentAge("Somchai", 20);
    std::cout << studentAge << '\n';

    Pair<int, double> coordinate(3, 4.5);
    std::cout << coordinate << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 pair.cpp -o pair
./pair
```

```
(Somchai, 20)
(3, 4.5)
```

### จุดที่ต้องสังเกต

- **`template <typename T1, typename T2>` ต้องเขียนซ้ำหน้า `operator<<`** เพราะ `operator<<`
  เป็น **free function** (ไม่ใช่ member function ของ `Pair`) และตัวมันเองก็เป็น function
  template ที่แยกจาก class template `Pair` — ทั้งสองต้องประกาศ template parameter ของตัวเอง
  แม้จะใช้ชื่อ `T1`, `T2` ซ้ำกันก็ตาม (ชื่อ parameter ไม่จำเป็นต้องตรงกันเป๊ะ แต่นิยมตั้งชื่อ
  ให้สอดคล้องกันเพื่อความเข้าใจง่าย)
- **`Pair<T1, T2>` ใน parameter ของ `operator<<`**: การเขียน `const Pair<T1, T2>& p` บอกว่า
  ฟังก์ชันนี้ทำงานกับ `Pair` ของ **type คู่ใดก็ได้** ไม่ใช่แค่ `Pair<std::string, int>` ตัวเดียว
  — เมื่อเราเขียน `std::cout << studentAge`, compiler จะ deduce `T1 = std::string, T2 = int`
  ให้อัตโนมัติจาก type ของ `studentAge`
- โครงสร้างนี้คล้ายกับ `std::pair<T1, T2>` ในไลบรารีมาตรฐาน (จาก `<utility>`) มาก — ในความเป็น
  จริง `std::pair` ก็ implement ด้วยแนวคิดเดียวกันนี้เป๊ะๆ เพียงแต่มีฟีเจอร์เสริมเพิ่มเติม เช่น
  `std::make_pair`, การเปรียบเทียบด้วย operator ต่างๆ, และ structured bindings (Part 72)

---

## 56.7 Non-Type Template Parameter: กำหนดค่าคงที่ตอน Compile-Time (Step 447)

Template Parameter ไม่จำเป็นต้องเป็น "ชนิดข้อมูล" เสมอไป — สามารถเป็น **ค่าคงที่** (เช่น
`int`, `std::size_t`, `bool`, enum, หรือ pointer) ที่รู้แน่นอนตอน compile-time ได้ด้วย เรียกว่า
**Non-Type Template Parameter**

ตัวอย่างคลาสสิกที่สุดคือการกำหนดขนาดของ array ให้เป็นส่วนหนึ่งของ type เอง:

```cpp
#include <cstddef>
#include <iostream>
#include <stdexcept>

// Non-type Template Parameter: N ไม่ใช่ "type" แต่เป็น "ค่าคงที่ที่รู้ตอน compile-time"
template <typename T, std::size_t N>
class FixedArray {
public:
    T& operator[](std::size_t index) {
        if (index >= N) {
            throw std::out_of_range("FixedArray::operator[]: index เกินขอบเขต");
        }
        return data_[index];
    }

    const T& operator[](std::size_t index) const {
        if (index >= N) {
            throw std::out_of_range("FixedArray::operator[]: index เกินขอบเขต");
        }
        return data_[index];
    }

    constexpr std::size_t size() const noexcept { return N; }

private:
    T data_[N] = {};   // N ต้องเป็นค่าคงที่ตอน compile-time เท่านั้นถึงจะประกาศ array แบบนี้ได้
};

int main() {
    FixedArray<int, 5> arr;   // N = 5 ถูกฝังเข้าไปใน type ตั้งแต่ compile-time
    for (std::size_t i = 0; i < arr.size(); ++i) {
        arr[i] = static_cast<int>(i * i);
    }
    for (std::size_t i = 0; i < arr.size(); ++i) {
        std::cout << arr[i] << ' ';
    }
    std::cout << '\n';
    std::cout << "size = " << arr.size() << '\n';

    // FixedArray<int, 5> กับ FixedArray<int, 10> เป็นคนละ type กันโดยสิ้นเชิง
    FixedArray<double, 3> smallArr;
    smallArr[0] = 1.1;
    smallArr[1] = 2.2;
    smallArr[2] = 3.3;
    std::cout << "smallArr size = " << smallArr.size() << '\n';

    try {
        smallArr[10] = 9.9;   // เกินขอบเขต -> throw
    } catch (const std::out_of_range& e) {
        std::cout << "จับ exception ได้ถูกต้อง: " << e.what() << '\n';
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 nontype.cpp -o nontype
./nontype
```

```
0 1 4 9 16 
size = 5
smallArr size = 3
จับ exception ได้ถูกต้อง: FixedArray::operator[]: index เกินขอบเขต
```

### ทำไม Non-Type Template Parameter ถึงมีประโยชน์

- **`T data_[N];` ต้องการให้ `N` เป็นค่าคงที่ตอน compile-time** — ถ้า `N` เป็นตัวแปรธรรมดา
  (parameter ปกติของ constructor เช่น) จะเขียน `T data_[N];` แบบ fixed-size array บน stack
  ไม่ได้เลย (จะกลายเป็น Variable Length Array ซึ่งไม่ใช่มาตรฐานของ C++) การส่ง `N` เป็น
  Non-Type Template Parameter ทำให้ compiler รู้ค่า `N` ตั้งแต่ตอน "รู้จัก type" คือรู้ก่อน
  แม้แต่จะสร้าง object จริงด้วยซ้ำ
- **`FixedArray<int, 5>` และ `FixedArray<int, 10>` เป็นคนละ type กันโดยสิ้นเชิง** — เหมือนกับ
  ที่ `Stack<int>` กับ `Stack<std::string>` เป็นคนละ type ใน 56.4 ผลที่ตามมาคือ**ไม่สามารถ
  เขียนฟังก์ชันที่รับ `FixedArray<T, N>` ขนาดใดก็ได้แบบง่ายๆ ได้** — ต้องทำเป็น template ที่มี
  `N` เป็น parameter ด้วยเสมอ (`template <typename T, std::size_t N> void f(FixedArray<T, N>&
  arr)`) นี่คือความแตกต่างสำคัญเมื่อเทียบกับการส่งขนาด array แบบ parameter ธรรมดาที่ยืดหยุ่นกว่า
  แต่ตรวจสอบขอบเขตได้แค่ตอน runtime
- **`std::array<T, N>` ของ STL ที่จะเรียนใน Part 59 ใช้แนวคิดเดียวกันนี้เป๊ะๆ** — เมื่อเราเขียน
  `std::array<int, 5>` เราก็กำลังใช้ Non-Type Template Parameter อยู่โดยไม่รู้ตัว ตอนนี้เมื่อ
  เข้าใจกลไกเบื้องหลังแล้ว การเรียนรู้ `std::array` ใน Part ถัดๆ ไปจะง่ายขึ้นมาก
- **ประโยชน์ด้าน Performance**: เพราะ `N` รู้ตอน compile-time, compiler สามารถ**ตรวจสอบขอบเขต
  บางกรณีตอน compile-time** และ**ทำ optimization ได้ดีกว่า** array ที่มีขนาดกำหนดตอน runtime
  (เช่น การ unroll loop) — นี่คือหลักการที่เรียกว่า **Zero-Cost Abstraction** ซึ่งเป็นหัวใจ
  สำคัญของปรัชญาการออกแบบภาษา C++ ทั้งภาษา

---

## 56.8 ทำไม Template ต้องอยู่ใน Header File (Step 448)

นี่คือกฎที่**สำคัญที่สุด**ของ Part นี้ และเป็นจุดที่มือใหม่เกือบทุกคนสะดุดอย่างน้อยหนึ่งครั้ง:
**Template implementation ต้อง visible อยู่ในทุก Translation Unit ที่เรียกใช้งานมัน** ซึ่งใน
ทางปฏิบัติหมายความว่า **ต้อง define ไว้ใน Header File เสมอ ไม่ใช่แยกไปไว้ใน `.cpp`**

### ทำไมถึงเป็นแบบนี้ — ย้อนกลับไปที่กระบวนการ Compile (ทบทวน Part 1 และ 17)

จำได้ไหมว่ากระบวนการ compile แต่ละไฟล์ `.cpp` (Translation Unit) เกิดขึ้น**แยกจากกันโดย
สมบูรณ์** — compiler แปลแต่ละ `.cpp` เป็น `.o` ทีละไฟล์ โดยไม่รู้จักเนื้อหาข้างในของ `.cpp`
ไฟล์อื่นเลย (รู้แค่ prototype ที่ประกาศไว้ใน header ที่ `#include` เข้ามา) จากนั้น **Linker**
จึงมาเชื่อมโยง `.o` ทุกไฟล์เข้าด้วยกันอีกที

ปัญหาของ Template คือ: **compiler ต้อง "เห็น" เนื้อหาเต็มๆ ของ template ตอน instantiate**
(ตอนที่มีการเรียกใช้งานจริง เช่น `myMax(3, 7)`) เพราะมันต้องรู้ว่าจะสร้างโค้ดสำหรับ `T = int`
อย่างไร ถ้า implementation ของ template อยู่ใน `.cpp` แยกไฟล์ (ที่ compile เป็น `.o` ไปแล้ว
โดยไม่รู้ว่าใครจะเรียกใช้ด้วย type อะไรบ้าง) จุดที่**เรียกใช้** template (ใน `.cpp` อีกไฟล์)
จะ**ไม่มีข้อมูลเพียงพอ**ที่จะสร้างโค้ดสำหรับ type นั้นได้เลย

### ลองทำให้เกิด Error จริงเพื่อดูให้เห็นภาพ

`mymath.h` — ประกาศ prototype อย่างเดียว (เหมือนวิธีที่เคยทำกับฟังก์ชันธรรมดาใน Part 17):

```cpp
#ifndef MYMATH_H
#define MYMATH_H

template <typename T>
T add(T a, T b);   // ประกาศ prototype อย่างเดียวใน header

#endif // MYMATH_H
```

`mymath.cpp` — implementation อยู่แยกไฟล์ (สไตล์ modular ปกติที่เคยใช้กับฟังก์ชันธรรมดา):

```cpp
#include "mymath.h"

// implementation อยู่ใน .cpp แยกต่างหาก -- นี่คือปัญหา
template <typename T>
T add(T a, T b) {
    return a + b;
}
```

`main.cpp`:

```cpp
#include <iostream>
#include "mymath.h"

int main() {
    std::cout << add(3, 4) << '\n';
    return 0;
}
```

คอมไพล์แยกแต่ละไฟล์ (เหมือนที่ Makefile ทำใน Part 55) แล้ว link เข้าด้วยกัน:

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 -c mymath.cpp -o mymath.o
g++ -Wall -Wextra -Wpedantic -std=c++17 -c main.cpp -o main.o
g++ mymath.o main.o -o prog
```

ผลลัพธ์:

```
/usr/bin/ld: main.o: in function `main':
main.cpp:(.text+0x13): undefined reference to `int add<int>(int, int)'
collect2: error: ld returned 1 exit status
```

สังเกตว่า**ทั้ง `mymath.cpp` และ `main.cpp` คอมไพล์ผ่านตอน `-c` โดยไม่มี error หรือ warning
เลย** — error เกิดขึ้นตอน **linking** เท่านั้น! เหตุผลคือ:

1. ตอน compile `mymath.cpp` เป็น `mymath.o`: compiler เห็น template `add` แต่**ไม่มีจุดไหน
   เรียกใช้งานมันเลยในไฟล์นี้** จึงไม่ instantiate อะไรทั้งสิ้น — `mymath.o` จึง**ไม่มีโค้ดของ
   `add<int>` อยู่ข้างในเลย** (ต่างจากตอนที่เราทำ `nm` ใน 56.5 ที่เจอ symbol เพราะมีการเรียกใช้
   งานอยู่ในไฟล์เดียวกัน)
2. ตอน compile `main.cpp` เป็น `main.o`: compiler เห็นการเรียก `add(3, 4)` และรู้ prototype
   จาก `mymath.h` แต่**ไม่เห็นเนื้อหาของ implementation เลย** (เพราะ header มีแค่ประกาศ
   prototype) จึง**สร้างโค้ดของ `add<int>` ให้ไม่ได้** — มันได้แต่ "จองที่ไว้" ว่าจะมีคนมาให้
   symbol `add<int>` ทีหลัง (คือฝากความหวังไว้กับ linker)
3. ตอน **linking**: linker ค้นหา symbol `int add<int>(int, int)` ในทุก `.o` ที่มี แต่**ไม่พบ
   ที่ไหนเลย** เพราะไม่มี `.o` ไฟล์ไหนเคย instantiate มันจริงๆ จึงเกิด **undefined reference**

### วิธีแก้ที่ถูกต้อง: ย้าย Implementation เข้า Header

```cpp
#ifndef MYMATH_FIXED_H
#define MYMATH_FIXED_H

template <typename T>
T add(T a, T b) {
    return a + b;
}

#endif // MYMATH_FIXED_H
```

```cpp
#include <iostream>
#include "mymath_fixed.h"

int main() {
    std::cout << add(3, 4) << '\n';
    std::cout << add(1.5, 2.5) << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 main_fixed.cpp -o main_fixed
./main_fixed
```

```
7
4
```

ตอนนี้ `main.cpp` เห็น implementation เต็มๆ ของ `add` ผ่าน `#include "mymath_fixed.h"` ทำให้
compiler สามารถ instantiate `add<int>` และ `add<double>` ได้ทันทีในไฟล์เดียวกับที่เรียกใช้งาน
— นี่คือเหตุผลที่**โค้ด template แทบทุกที่ในโลกจริง (รวมถึง STL ทั้งหมด) ถูกเขียนอยู่ในไฟล์
header เท่านั้น** สังเกตได้จากการที่ `<vector>`, `<map>`, `<algorithm>` ที่เรากำลังจะใช้งานใน
Part ถัดๆ ไปล้วนเป็น header ล้วนๆ (header-only) ไม่มีไฟล์ `.cpp` แยกให้ compile เลย

### ทางเลือกขั้นสูง: Explicit Instantiation (รู้ไว้ แต่ไม่แนะนำให้ใช้เป็นค่าเริ่มต้น)

มีอีกวิธีหนึ่งที่ทำให้แยก implementation ไป `.cpp` ได้ เรียกว่า **Explicit Instantiation** —
บังคับให้ compiler สร้างโค้ดสำหรับ type ที่ระบุไว้ล่วงหน้าตรงๆ ใน `.cpp` นั้นเลย:

```cpp
#include "mymath.h"

template <typename T>
T add(T a, T b) {
    return a + b;
}

// Explicit Instantiation: บังคับให้ compiler สร้างโค้ดของ add<int> ไว้ในไฟล์ .o นี้ตรงๆ
template int add<int>(int, int);
```

วิธีนี้**ใช้งานได้จริง** (compile และ link ผ่านสำหรับ `add(3, 4)` ที่เป็น `int`) แต่มีข้อเสีย
ใหญ่คือ **ใช้ได้เฉพาะกับ type ที่เรารู้ล่วงหน้าและเขียน explicit instantiation ไว้ให้ครบเท่านั้น**
— ถ้าผู้ใช้ header เรียก `add(1.5, 2.5)` (เป็น `double`) โดยที่ไม่มีบรรทัด
`template double add<double>(double, double);` เขียนไว้ใน `.cpp` จะเจอ undefined reference
แบบเดิมทันที เทคนิคนี้เหมาะกับกรณีพิเศษที่รู้แน่ชัดว่า template จะถูกใช้กับ type ที่จำกัดตายตัว
เท่านั้น (เช่น library ภายในองค์กรที่ควบคุมการใช้งานได้) แต่**ไม่เหมาะกับ template ทั่วไปที่
ต้องการให้ใช้ได้กับ type ใดก็ได้** — ด้วยเหตุนี้ **แนวทางมาตรฐานและปลอดภัยที่สุดคือ implement
template ทั้งหมดไว้ใน header เสมอ** ตลอดหลักสูตรนี้เราจะยึดหลักการนี้ทุกครั้งที่เขียน template

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **แยก implementation ของ template ไปไว้ใน `.cpp`** — ดังที่สาธิตใน 56.8 จะได้ **linker
   error `undefined reference`** ที่หน้าตาสับสนมาก (บอกว่า "undefined reference" ทั้งที่เรา
   เห็น implementation อยู่ในตาเปล่าใน `.cpp` อีกไฟล์) วิธีแก้คือย้าย implementation ทั้งหมด
   เข้า header เสมอ
2. **สับสนระหว่าง Template Argument Deduction ล้มเหลว กับ compile error ทั่วไป** —
   ข้อความ error ของ template มักจะยาวและซับซ้อนกว่า error ปกติมาก (โดยเฉพาะกับ template ที่
   ซ้อนกันหลายชั้น) เวลาเจอ error แบบนี้ ให้อ่านจาก**บรรทัดแรกสุด**ของ error message ก่อนเสมอ
   ซึ่งมักจะบอกจุดที่เรียกใช้งานจริงที่ทำให้เกิดปัญหา
3. **ส่ง argument คนละ type ให้ Template Parameter ตัวเดียวกัน** — เช่น `myMax(10, 3.5)` เมื่อ
   `T myMax(T a, T b)` ผูก `a` กับ `b` เป็น `T` ตัวเดียวกัน จะเกิด deduction ambiguity ทันที
   ต้อง cast ให้ตรงชนิดกันเอง หรือระบุ `myMax<double>(...)` ชัดเจน
4. **ลืมว่า `Stack<int>` กับ `Stack<double>` เป็นคนละ type กันโดยสิ้นเชิง** — พยายามส่ง
   `Stack<int>` ให้ฟังก์ชันที่รับ `Stack<double>&` โดยตรงจะ compile error ทันที ต้องเขียน
   ฟังก์ชันเป็น template เองด้วย (`template <typename T> void f(Stack<T>& s)`) ถ้าต้องการให้
   ทำงานกับ `Stack<T>` ชนิดใดก็ได้
5. **ใช้ non-type template parameter ที่ไม่ใช่ compile-time constant** — เช่น พยายามเขียน
   `FixedArray<int, size>` โดยที่ `size` เป็นตัวแปรธรรมดาที่ค่าจะรู้ตอน runtime เท่านั้น
   (เช่นอ่านจาก `std::cin`) จะ compile error ทันที เพราะ non-type template parameter ต้องเป็น
   ค่าคงที่ที่ compiler รู้ได้ตอน compile-time เท่านั้น (`const`/`constexpr` หรือ literal ตรงๆ)
6. **ลืม `#include` guard หรือ `#pragma once` ใน header ที่มี template** — เหมือนกับ header
   ทั่วไป (ทบทวน Part 17) ถ้า header ที่มี template ถูก `#include` ซ้ำในไฟล์เดียวกัน (ผ่านสาย
   การ include ที่ซับซ้อน) โดยไม่มี include guard จะเกิด "redefinition" error — กฎการใช้
   `#ifndef`/`#define`/`#endif` หรือ `#pragma once` ยังคงสำคัญเหมือนเดิมทุกประการ

---

## แบบฝึกหัดท้ายบท

1. เขียน function template `myMin(T a, T b)` ที่คืนค่าที่**น้อยกว่า**ระหว่างสองค่า (คล้าย
   `myMax` แต่กลับเงื่อนไข) ทดสอบกับ `int`, `double`, และ `std::string`
2. เขียน function template `mySwap(T& a, T& b)` ที่สลับค่าของตัวแปรสองตัวที่รับเข้ามาแบบ
   reference (ไม่ return ค่าอะไร แก้ไข `a` กับ `b` ตรงๆ) ทดสอบกับ `int` และ `std::string`
3. เขียน class template `Queue<T>` (คิวแบบ FIFO — First In First Out) ที่มี method
   `enqueue(const T&)`, `dequeue()`, `front()`, `empty()`, `size()` (ใช้ `std::deque<T>` เป็น
   ตัวเก็บข้อมูลข้างในเหมือนที่ `Stack<T>` ใช้ `std::vector<T>`)
4. เขียน class template `Triple<T1, T2, T3>` ที่เก็บค่า 3 ค่าคนละ type กันได้ (ต่อยอดจาก
   `Pair<T1, T2>` ใน 56.6) พร้อม `operator<<` แสดงผลในรูปแบบ `(a, b, c)`
5. อธิบายด้วยคำพูดของตัวเอง (ไม่ต้องเขียนโค้ด) ว่าทำไมการเขียน
   `template <typename T> T add(T a, T b);` ไว้ใน header แล้ว implement ใน `.cpp` แยก จะทำให้
   เกิด linker error แม้ทั้งสองไฟล์จะ compile ผ่านแยกกันได้ปกติก็ตาม
6. เขียน class template `FixedRingBuffer<T, N>` (ring buffer ขนาดคงที่) ที่มี method
   `push(const T&)` (เขียนทับตำแหน่งเก่าเมื่อเต็ม) และ `operator[](std::size_t)` โดยใช้
   non-type template parameter `N` กำหนดขนาด (คำใบ้: ใช้ `index % N` ในการคำนวณตำแหน่งจริง)

### แนวทางเฉลยข้อ 2: `mySwap`

```cpp
#include <iostream>
#include <string>
#include <utility>

template <typename T>
void mySwap(T& a, T& b) {
    T temp = std::move(a);
    a = std::move(b);
    b = std::move(temp);
}

int main() {
    int x = 1, y = 2;
    mySwap(x, y);
    std::cout << "x=" << x << " y=" << y << '\n';

    std::string s1 = "ต้นทาง";
    std::string s2 = "ปลายทาง";
    mySwap(s1, s2);
    std::cout << "s1=" << s1 << " s2=" << s2 << '\n';

    return 0;
}
```

ผลลัพธ์:

```
x=2 y=1
s1=ปลายทาง s2=ต้นทาง
```

จุดสำคัญ: `mySwap` รับ argument เป็น **reference** (`T&`) ไม่ใช่ pass by value เหมือน `myMax`
เพราะจุดประสงค์คือ**แก้ไขค่าตัวแปรต้นฉบับโดยตรง** ไม่ใช่คำนวณค่าใหม่แล้ว return — ถ้าใช้
pass by value การสลับค่าจะเกิดขึ้นแค่กับสำเนาข้างในฟังก์ชัน แล้วหายไปทันทีที่ฟังก์ชันจบการทำงาน
(ทบทวนแนวคิด pass by value vs pass by reference จาก Part 6 และ Part 43) นอกจากนี้ยังใช้
`std::move` (เกริ่นนำจาก Part 46, จะเรียนเต็มรูปแบบใน Part 70) เพื่อย้ายค่าแทนการ copy
ซึ่งมีประสิทธิภาพดีกว่าอย่างมากสำหรับ type ที่ copy แพง เช่น `std::string`

### แนวทางเฉลยข้อ 3: `Queue<T>`

```cpp
#include <deque>
#include <iostream>
#include <stdexcept>
#include <string>

template <typename T>
class Queue {
public:
    void enqueue(const T& value) {
        data_.push_back(value);
    }

    void dequeue() {
        if (empty()) {
            throw std::out_of_range("Queue::dequeue: queue ว่างเปล่า");
        }
        data_.pop_front();
    }

    T& front() {
        if (empty()) {
            throw std::out_of_range("Queue::front: queue ว่างเปล่า");
        }
        return data_.front();
    }

    bool empty() const noexcept { return data_.empty(); }
    std::size_t size() const noexcept { return data_.size(); }

private:
    std::deque<T> data_;
};

int main() {
    Queue<std::string> q;
    q.enqueue("คนที่ 1");
    q.enqueue("คนที่ 2");
    q.enqueue("คนที่ 3");

    std::cout << "คิวแรกสุด: " << q.front() << " (ทั้งหมด " << q.size() << " คน)\n";
    q.dequeue();
    std::cout << "หลัง dequeue: " << q.front() << " (ทั้งหมด " << q.size() << " คน)\n";

    Queue<int> emptyQ;
    try {
        emptyQ.dequeue();
    } catch (const std::out_of_range& e) {
        std::cout << "จับ exception ได้ถูกต้อง: " << e.what() << '\n';
    }

    return 0;
}
```

ผลลัพธ์:

```
คิวแรกสุด: คนที่ 1 (ทั้งหมด 3 คน)
หลัง dequeue: คนที่ 2 (ทั้งหมด 2 คน)
จับ exception ได้ถูกต้อง: Queue::dequeue: queue ว่างเปล่า
```

จุดสำคัญ: `Queue<T>` เลือกใช้ `std::deque<T>` แทน `std::vector<T>` เป็นตัวเก็บข้อมูลข้างใน
เพราะ `enqueue` เพิ่มที่ท้าย (`push_back`) แต่ `dequeue` ต้องลบที่**หัว** (`pop_front`) —
`std::vector` ทำ `pop_front` ได้ไม่มีประสิทธิภาพเลย (ต้องขยับข้อมูลทั้งหมดไปข้างหน้า 1 ตำแหน่ง
เป็น O(n)) ในขณะที่ `std::deque` ออกแบบมาให้เพิ่ม/ลบได้เร็วทั้งหัวและท้ายเป็น O(1) เรื่องนี้จะ
อธิบายอย่างละเอียดเมื่อเปรียบเทียบ container ต่างๆ ใน **Part 58** และ **Part 60**

---

## สรุปท้ายบท

ใน Part นี้เราได้เปิดประตูสู่โลกของ **Generic Programming** ซึ่งเป็นรากฐานสำคัญที่สุดของ
Module E:

- เข้าใจปัญหาที่ Template แก้ไข: การเขียนโค้ดซ้ำๆ สำหรับแต่ละชนิดข้อมูล และทำไม Template
  ปลอดภัยและทรงพลังกว่า Macro หรือ Function Overloading ธรรมดา
- เขียน **Function Template** ด้วย `template <typename T>` และเข้าใจกลไก
  **Template Argument Deduction** อย่างละเอียด รวมถึงกรณีที่ deduction ล้มเหลว
- เขียน **Class Template** ของตัวเอง (`Stack<T>`) ที่ทำงานได้กับชนิดข้อมูลใดก็ได้ โดยยังคง
  type-safe เต็มรูปแบบ
- พิสูจน์ด้วยเครื่องมือจริง (`nm`) ว่า **Template Instantiation** เกิดขึ้นตอน compile-time
  จริงๆ — compiler สร้างโค้ดแยกสำหรับทุกชนิดข้อมูลที่ถูกใช้งาน
- เขียน Template ที่มี **หลาย Template Parameter** (`Pair<T1, T2>`) และ
  **Non-Type Template Parameter** (`FixedArray<T, N>`) ซึ่งเป็นรากฐานของ `std::array` ที่จะ
  เรียนใน Part 59
- เข้าใจอย่างลึกซึ้งว่า**ทำไม Template ต้อง implement อยู่ใน Header เสมอ** พร้อมพิสูจน์ด้วย
  linker error จริง และรู้จักทางเลือกขั้นสูงอย่าง Explicit Instantiation

Template ที่เราเขียนใน Part นี้ยังเป็นแบบ **"generic เท่ากันหมดทุก type"** — แต่ในโลกจริงบางครั้ง
เราต้องการให้ template มีพฤติกรรม**พิเศษ**สำหรับบาง type โดยเฉพาะ (เช่น `const char*` ที่
เปรียบเทียบด้วย `==` ตรงๆ ไม่ได้ความหมายที่ถูกต้อง) และบางครั้งเราต้องการฟังก์ชันที่รับ
argument **กี่ตัวก็ได้** ไม่จำกัดจำนวน — ทั้งสองเรื่องนี้คือหัวข้อของ **Part 57**:
**Template Specialization และ Variadic Template**

**ต่อไป:** [Part 57 — Template Specialization และ Variadic Template](./part-057-template-specialization.md)
