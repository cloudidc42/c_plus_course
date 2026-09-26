# Part 64: Function Object, Lambda, std::function (Step 505–512)

> Module E — Templates, Generic Programming และ STL | Part 64 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 505–512
> Part ก่อนหน้า: [Part 63 — STL Algorithm แบบเจาะลึก](./part-063-stl-algorithms.md) | Part ถัดไป: [Part 65 — std::string และ string_view/Regex](./part-065-string-advanced.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Function Object (Functor)** คืออะไร และเขียน class ที่
   overload `operator()` เพื่อทำให้ object เรียกใช้งานได้เหมือนฟังก์ชัน
2. อธิบายได้ว่าทำไม Functor ยังมีประโยชน์อยู่แม้ในยุคที่ Lambda มีให้ใช้แล้ว
   โดยเฉพาะความสามารถในการเก็บ **state** ระหว่างการเรียกใช้
3. เขียน Lambda Expression ในรูปแบบเต็ม `[capture](params) -> return_type { body }`
   ได้อย่างคล่องแคล่ว
4. เข้าใจความแตกต่างระหว่าง Capture by value (`[=]`) กับ Capture by reference
   (`[&]`) และเลือกใช้ได้อย่างเหมาะสมและปลอดภัย
5. ใช้คีย์เวิร์ด `mutable` เพื่ออนุญาตให้ Lambda แก้ไขสำเนาของตัวแปรที่ capture
   by value ได้
6. เขียน **Generic Lambda** (C++14) ที่ใช้ `auto` เป็น parameter type เพื่อรับ
   ข้อมูลได้หลายชนิดโดยไม่ต้องเขียน Lambda ซ้ำหลายตัว
7. ใช้ `std::function` เก็บ Callable object ใดๆ ก็ตาม (function pointer,
   functor, lambda) ไว้ในตัวแปรชนิดเดียวกันได้
8. ออกแบบระบบ **Callback** ด้วย `std::function` ในสไตล์ที่ใช้จริงในโปรแกรม
   ระดับ Production

---

## 64.1 Function Object (Functor) คืออะไร (Step 505)

**Function Object** หรือที่เรียกสั้นๆ ว่า **Functor** คือ object ของ class ที่
**overload `operator()`** (เรียนวิธี overload operator มาแล้วใน Part 51) ทำให้
สามารถ "เรียก" object นั้นด้วยวงเล็บ `()` เหมือนกับเรียกฟังก์ชันธรรมดาได้ทันที:

```cpp
#include <cstdio>

class Multiplier {
public:
    explicit Multiplier(int factor) : factor_(factor) {}

    int operator()(int x) const { return x * factor_; }

private:
    int factor_;
};

int main(void) {
    Multiplier times3(3);
    std::printf("%d\n", times3(10)); // เรียก object เหมือนฟังก์ชัน -> 30

    Multiplier times5(5);
    std::printf("%d\n", times5(10)); // 50

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 functor_basic.cpp -o functor_basic
./functor_basic
# 30
# 50
```

สิ่งที่เกิดขึ้นตรง `times3(10)` จริงๆ แล้ว compiler แปลงเป็น
`times3.operator()(10)` เบื้องหลัง — `times3` ไม่ใช่ฟังก์ชัน แต่เป็น **object**
ธรรมดาของ class `Multiplier` ที่บังเอิญเรียกด้วยวงเล็บได้เหมือนฟังก์ชัน จุดที่
ทำให้ Functor ต่างจากฟังก์ชันธรรมดาอย่างสิ้นเชิงคือ **Functor มี state ภายในตัว
มันเอง** — `Multiplier times3(3)` กับ `Multiplier times5(5)` เป็น object คนละ
ตัวที่เก็บค่า `factor_` ต่างกัน (3 กับ 5) แม้จะใช้ class เดียวกันก็ตาม ทำให้
เขียนฟังก์ชัน "ที่ปรับแต่งได้" (parameterized function) ได้โดยไม่ต้องใช้ Global
Variable หรือส่งพารามิเตอร์เพิ่มทุกครั้งที่เรียก

**เหตุผลที่มาตรฐาน STL ใช้คำว่า "Function Object" แทน "ฟังก์ชัน" ตรงๆ** เพราะ
algorithm หลายตัวใน `<algorithm>` (ที่เรียนใน Part 63) ต้องการรับพารามิเตอร์ที่
"เรียกได้เหมือนฟังก์ชัน" (เรียกรวมๆ ว่า **Callable**) ซึ่งอาจเป็นได้ทั้งฟังก์ชัน
ธรรมดา (function pointer), Functor, หรือ Lambda Expression — ทั้งสามแบบล้วน
เรียกด้วย `()` ได้เหมือนกันหมด แม้จะมีธรรมชาติภายในต่างกันโดยสิ้นเชิง

---

## 64.2 ทำไม Functor ยังสำคัญแม้จะมี Lambda แล้ว (Step 506)

Lambda Expression (ที่จะเรียนในหัวข้อ 64.3) ถือกำเนิดใน C++11 และกลายเป็นวิธี
เขียน Callable ที่นิยมที่สุดในโค้ดสมัยใหม่ (ดังที่เห็นตลอด Part 63) แต่ Functor
ก็ยังไม่ได้ล้าสมัยไปเลย เพราะมีจุดเด่นเฉพาะตัวที่ Lambda ไม่สามารถทดแทนได้ดี
เท่าในบางสถานการณ์:

| คุณสมบัติ | Functor (class + `operator()`) | Lambda Expression |
|---|---|---|
| การเก็บ State ที่แก้ไขได้และอ่านค่ากลับได้ง่าย | ทำได้เป็นธรรมชาติ (เป็น member variable ปกติ) | ทำได้ (ผ่าน capture) แต่เข้าถึง state จากภายนอกหลังเรียกเสร็จยุ่งยากกว่า |
| นำกลับมาใช้ซ้ำได้หลายที่ในโค้ด | ดีมาก — ประกาศ class ครั้งเดียว เรียกใช้ได้ทั่วโปรแกรม | ต้องเขียน lambda ซ้ำทุกจุดที่ใช้ (เว้นแต่เก็บใน `auto`/`std::function` ไว้ก่อน) |
| เขียนโค้ดสั้นสำหรับ logic ใช้ครั้งเดียว | เขียนยาว ต้องประกาศ class แยก | สั้นและกระชับมาก เหมาะกับ inline logic |
| Overload หลายรูปแบบ operator() ใน object เดียว | ทำได้ (เขียนหลาย overload ของ `operator()`) | ทำไม่ได้ (Lambda มี `operator()` แบบเดียว) |
| ใช้เป็น Template parameter ที่ต้อง instantiate ซ้ำได้ชัดเจน | ดี — ชื่อ class ชัดเจน อ่านง่ายใน error message | error message จาก Lambda มักอ่านยากกว่า (ชื่อ type เป็น compiler-generated) |

ตัวอย่างการใช้ Functor เพื่อ **สะสม state** ระหว่างการเรียกซ้ำผ่าน algorithm —
สิ่งที่ฟังก์ชันธรรมดา (function pointer) ทำไม่ได้เลยเพราะไม่มีที่เก็บข้อมูล
ระหว่างการเรียกแต่ละครั้ง:

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>

// functor ที่ "จำ" สถานะได้ระหว่างการเรียกซ้ำๆ ผ่าน std::for_each — จุดเด่นที่ฟังก์ชันธรรมดาทำไม่ได้
class CountEven {
public:
    int count = 0;

    void operator()(int x) {
        if (x % 2 == 0) {
            ++count;
        }
    }
};

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};

    CountEven counter;
    // std::for_each คืน functor กลับมาให้ (หลังใช้งานจบ) เราจึงอ่านสถานะที่สะสมไว้ได้
    counter = std::for_each(v.begin(), v.end(), counter);
    std::printf("จำนวนเลขคู่: %d\n", counter.count);

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 functor_state.cpp -o functor_state
./functor_state
# จำนวนเลขคู่: 4
```

`std::for_each` เมื่อทำงานเสร็จจะ **คืน (return) functor ที่ถูกส่งเข้าไป
กลับมาให้** (สังเกตว่าเป็นการคืนค่าแบบสำเนา ไม่ใช่ reference) เราจึงต้องเขียน
`counter = std::for_each(...)` เพื่อรับสำเนาที่มีค่า `count` สะสมล่าสุดกลับมา
เก็บไว้ในตัวแปร `counter` เดิม (ถ้าเรียก `std::for_each(v.begin(), v.end(),
counter);` เฉยๆ โดยไม่รับค่ากลับมา `counter.count` ที่ตัวแปรต้นฉบับจะยังเป็น 0
อยู่เหมือนเดิม เพราะ functor ถูกส่งเข้า `std::for_each` แบบ **สำเนา** ไม่ใช่
reference) นี่คือจุดที่ต้องระวังเวลาใช้ Functor ที่มี state ร่วมกับ algorithm
ของ STL — ถ้าไม่รับค่าที่ algorithm คืนกลับมา state ที่สะสมไว้จะหายไปเมื่อ
algorithm ทำงานจบ

---

## 64.3 Lambda Expression แบบละเอียด (Step 507)

**Lambda Expression** คือวิธีเขียนฟังก์ชันแบบไม่มีชื่อ (anonymous function)
ตรงจุดที่ต้องใช้งานเลย โดยไม่ต้องประกาศฟังก์ชันหรือ class แยกไว้ที่อื่นก่อน
รูปแบบเต็มของ Lambda คือ:

```
[capture](parameters) -> return_type { body }
```

- **`[capture]`** — บอกว่า Lambda จะ "จับ" ตัวแปรจาก scope ภายนอกมาใช้อย่างไร
  (รายละเอียดในหัวข้อ 64.4)
- **`(parameters)`** — พารามิเตอร์ เหมือนฟังก์ชันปกติทุกประการ
- **`-> return_type`** — ชนิดข้อมูลที่คืนค่า (เป็นทางเลือก — ถ้า compiler เดา
  ได้เองจาก `return` statement ภายในตัวมัน ก็ไม่จำเป็นต้องเขียน)
- **`{ body }`** — เนื้อหาของฟังก์ชัน เหมือนฟังก์ชันปกติทุกประการ

```cpp
#include <cstdio>

int main(void) {
    // รูปแบบเต็ม: [capture](parameters) -> return_type { body }
    auto add = [](int a, int b) -> int {
        return a + b;
    };
    std::printf("%d\n", add(3, 4));

    // ถ้า compiler เดาชนิดข้อมูลคืนค่าได้เองจาก return statement ไม่ต้องเขียน -> return_type
    auto square = [](int x) {
        return x * x;
    };
    std::printf("%d\n", square(5));

    // lambda ที่ไม่รับพารามิเตอร์และไม่ capture อะไรเลย
    auto greet = []() {
        std::printf("Hello from lambda!\n");
    };
    greet();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 lambda_syntax.cpp -o lambda_syntax
./lambda_syntax
# 7
# 25
# Hello from lambda!
```

ตัวแปร `add`, `square`, `greet` ที่ประกาศด้วย `auto` มีชนิดข้อมูลจริงๆ เป็น
**closure type** ซึ่งเป็นชนิดข้อมูลที่ compiler สร้างขึ้นมาให้อัตโนมัติ (ไม่มีชื่อ
ที่มนุษย์เขียนถึงได้ตรงๆ) เบื้องหลัง Lambda แต่ละตัวคือการสร้าง class ที่มี
`operator()` ให้อัตโนมัติ — พูดง่ายๆ คือ **Lambda คือ Functor ที่ compiler เขียน
class ให้เราแบบอัตโนมัติ** นี่คือเหตุผลที่ทำให้ Lambda สามารถใช้แทน Functor ได้
ในเกือบทุกกรณีที่ไม่ต้องการนำกลับมาใช้ซ้ำหลายที่หรือไม่ต้องการ overload
`operator()` หลายรูปแบบ

`[]` เปล่าๆ (ไม่ capture อะไรเลย) หมายความว่า Lambda ตัวนั้นเข้าถึงได้แค่
พารามิเตอร์ของตัวมันเองและตัวแปร/ฟังก์ชัน global เท่านั้น เข้าถึงตัวแปร local
ของ scope ที่ล้อมรอบไม่ได้เลย — ถ้าต้องการเข้าถึงตัวแปรภายนอกต้อง capture
เข้ามาก่อน ซึ่งเป็นหัวข้อถัดไป

---

## 64.4 Capture by Value vs Capture by Reference (Step 508)

ส่วน `[capture]` ของ Lambda มีรูปแบบหลักๆ ดังนี้:

| รูปแบบ | ความหมาย |
|---|---|
| `[]` | ไม่ capture ตัวแปรใดเลย |
| `[x]` | capture ตัวแปร `x` **by value** (คัดลอกค่า ณ ตอนสร้าง lambda) |
| `[&x]` | capture ตัวแปร `x` **by reference** (อ้างอิงตัวแปรจริง ไม่คัดลอก) |
| `[=]` | capture ตัวแปรทั้งหมดที่ใช้ในตัว lambda **by value** โดยอัตโนมัติ |
| `[&]` | capture ตัวแปรทั้งหมดที่ใช้ในตัว lambda **by reference** โดยอัตโนมัติ |
| `[x, &y]` | capture ผสม: `x` by value, `y` by reference |
| `[=, &y]` | capture ทุกตัวแปร by value ยกเว้น `y` ที่ capture by reference |

```cpp
#include <cstdio>

int main(void) {
    int x = 10;

    auto by_value = [x]() {
        std::printf("by_value: x = %d\n", x);
    };
    auto by_ref = [&x]() {
        std::printf("by_ref: x = %d\n", x);
    };

    x = 99;
    by_value(); // ยังพิมพ์ 10 เพราะ copy ค่า x ไปตั้งแต่ตอนสร้าง lambda
    by_ref();   // พิมพ์ 99 เพราะอ้างอิงตัวแปรจริง ไม่ใช่สำเนา

    int a = 1, b = 2, c = 3;

    auto capture_all_by_value = [=]() {
        std::printf("a+b+c = %d\n", a + b + c);
    };
    auto capture_all_by_ref = [&]() {
        a += 100;
        std::printf("a หลังแก้ไขภายใน lambda = %d\n", a);
    };

    capture_all_by_value();
    capture_all_by_ref();
    std::printf("a ใน main หลังเรียก lambda = %d\n", a); // เปลี่ยนจริง เพราะ capture by reference

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 capture_demo.cpp -o capture_demo
./capture_demo
```

ผลลัพธ์:

```
by_value: x = 10
by_ref: x = 99
a+b+c = 6
a หลังแก้ไขภายใน lambda = 101
a ใน main หลังเรียก lambda = 101
```

จุดสำคัญที่ต้องเข้าใจให้แม่นคือ **ค่าที่ capture by value จะถูกคัดลอก ณ ตอนที่
สร้าง lambda (ตอนที่ compiler เจอบรรทัด `[x]() {...}`) ไม่ใช่ตอนที่เรียก lambda
นั้นทำงาน** — ในตัวอย่าง `by_value` ถูกสร้างตอน `x` ยังมีค่า `10` แม้จะไปแก้
`x = 99` หลังจากนั้น แล้วค่อยเรียก `by_value()` ผลลัพธ์ก็ยังเป็น `10` เหมือนเดิม
เพราะสำเนาที่เก็บอยู่ใน closure object ของ `by_value` ไม่ได้เชื่อมโยงกับตัวแปร
`x` ตัวจริงอีกต่อไปแล้ว ต่างจาก `by_ref` ที่เก็บแค่ **reference** ไปยัง `x`
ตัวจริง เมื่อ `x` เปลี่ยนค่า การเรียก `by_ref()` จึงเห็นค่าล่าสุดเสมอ

**หลักการเลือกใช้**: ใช้ `[=]`/capture by value เมื่อต้องการให้ lambda "แช่แข็ง"
ค่า ณ เวลาที่สร้างไว้ (ปลอดภัยกว่า ไม่มีปัญหาเรื่องอายุของตัวแปรต้นทาง) ใช้
`[&]`/capture by reference เมื่อต้องการให้ lambda เห็นค่าล่าสุดเสมอ หรือต้องการ
แก้ไขตัวแปรภายนอกจริงๆ ผ่าน lambda — แต่การ capture by reference มีความเสี่ยง
สำคัญที่ต้องระวังมาก ซึ่งจะพูดถึงใน Common Pitfalls ท้ายบท

---

## 64.5 Mutable Lambda (Step 509)

โดย default `operator()` ที่ compiler สร้างให้ Lambda เป็น **`const`** เสมอ
นั่นแปลว่าตัวแปรที่ capture by value เข้ามาจะถูกมองว่าเป็น `const` ภายในตัว
lambda ไปด้วย แก้ไขค่าของมันไม่ได้ ถ้าต้องการอนุญาตให้แก้ไข **สำเนา** ของ
ตัวแปรที่ capture by value ได้ ต้องใส่คีย์เวิร์ด **`mutable`** หลังวงเล็บ
พารามิเตอร์:

```cpp
#include <cstdio>

int main(void) {
    int counter = 0;

    // capture by value ปกติจะแก้ไข "สำเนา" ภายใน lambda ไม่ได้ (operator() เป็น const โดย default)
    // ต้องใส่ mutable เพื่ออนุญาตให้แก้ไขสำเนาที่เก็บอยู่ภายใน lambda object ได้
    auto increment = [counter]() mutable {
        ++counter;
        std::printf("ภายใน lambda: counter = %d\n", counter);
    };

    increment();
    increment();
    increment();

    std::printf("ภายนอก lambda: counter = %d\n", counter); // ยังเป็น 0 เพราะแก้แค่สำเนาของ lambda

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 mutable_lambda.cpp -o mutable_lambda
./mutable_lambda
```

ผลลัพธ์:

```
ภายใน lambda: counter = 1
ภายใน lambda: counter = 2
ภายใน lambda: counter = 3
ภายนอก lambda: counter = 0
```

สังเกตว่าค่า `counter` ที่พิมพ์จากภายใน lambda **สะสมเพิ่มขึ้นเรื่อยๆ** ในแต่ละ
ครั้งที่เรียก `increment()` (1, 2, 3) เพราะ closure object ของ `increment`
เก็บสำเนาของ `counter` ไว้เป็น **member variable ถาวร** ของมันเอง (ไม่ได้สร้าง
สำเนาใหม่ทุกครั้งที่เรียก) การใส่ `mutable` แค่อนุญาตให้แก้ไขสำเนานั้นได้เท่านั้น
— แต่ `counter` ตัวจริงใน `main()` ยังคงเป็น `0` เหมือนเดิมเสมอ เพราะไม่มีการ
เชื่อมโยงกันเลยตั้งแต่ตอน capture by value

---

## 64.6 Generic Lambda (C++14) (Step 510)

ตั้งแต่ C++14 เป็นต้นมา Lambda สามารถใช้ **`auto`** เป็นชนิดข้อมูลของพารามิเตอร์
ได้ เรียกว่า **Generic Lambda** — compiler จะสร้าง `operator()` แบบ template
ให้อัตโนมัติเบื้องหลัง ทำให้ Lambda ตัวเดียวใช้ได้กับหลายชนิดข้อมูล คล้ายกับ
Function Template ที่เรียนใน Part 56:

```cpp
#include <cstdio>
#include <string>

int main(void) {
    // generic lambda (C++14): ใช้ auto แทน type ของ parameter ได้
    // compiler จะ generate operator() แบบ template ให้อัตโนมัติ
    auto print_twice = [](auto x) {
        auto s = std::to_string(x);
        std::printf("%s %s\n", s.c_str(), s.c_str());
    };
    print_twice(5);
    print_twice(3.14);

    auto add = [](auto a, auto b) {
        return a + b;
    };
    std::printf("%d\n", add(2, 3));
    std::printf("%f\n", add(2.5, 1.5));

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 generic_lambda.cpp -o generic_lambda
./generic_lambda
```

ผลลัพธ์:

```
5 5
3.140000 3.140000
5
4.000000
```

`print_twice` ตัวเดียวใช้ได้ทั้งกับ `int` (5) และ `double` (3.14) โดยไม่ต้อง
เขียนแยก และ `add` ก็เช่นกัน — เรียก `add(2, 3)` ได้ผล `int` (5) และเรียก
`add(2.5, 1.5)` ได้ผล `double` (4.0) โดยใช้โค้ดตัวเดียวกันทั้งหมด เบื้องหลัง
เมื่อ Lambda ถูกเรียกด้วยชนิดข้อมูลต่างกัน compiler จะ instantiate `operator()`
แบบ template ขึ้นมาใหม่แยกกันสำหรับแต่ละชนิดข้อมูลที่ถูกใช้จริง (เหมือนกับ
Function Template ทุกประการ) ทำให้ยังคง type-safe และมีประสิทธิภาพเทียบเท่า
การเขียนฟังก์ชันแยกสำหรับแต่ละชนิดข้อมูลด้วยมือ

Generic Lambda มีประโยชน์มากเมื่อใช้เป็น comparator หรือ predicate ให้กับ
algorithm ของ STL (Part 63) ที่ต้องการรองรับ container หลายชนิด โดยไม่ต้อง
เขียน lambda แยกสำหรับแต่ละชนิดข้อมูลซ้ำไปมา

---

## 64.7 std::function: เก็บ Callable ใดๆ ไว้ในตัวแปรเดียว (Step 511)

ปัญหาหนึ่งของ Lambda คือ **แต่ละตัวมี closure type ของตัวเองที่ไม่เหมือนกันเลย
แม้จะมี signature เหมือนกัน** (รับพารามิเตอร์และคืนค่าแบบเดียวกัน) ทำให้เขียน
โค้ดที่ต้อง "เปลี่ยน" lambda ที่เก็บอยู่ในตัวแปรเดียวกัน หรือเก็บ lambda ไว้ใน
`std::vector` ที่ต้องมีชนิดข้อมูลเดียวกันทำได้ยาก — **`std::function`**
(อยู่ใน header `<functional>`) คือคำตอบของปัญหานี้ มันคือ **Type Erasure
Wrapper** ที่เก็บ Callable **ชนิดใดก็ได้** (function pointer, functor,
lambda) ไว้ในตัวแปรเดียวกัน ตราบใดที่ signature (พารามิเตอร์และ return type)
ตรงกัน:

```cpp
#include <cstdio>
#include <functional>

int plain_add(int a, int b) {
    return a + b;
}

class Adder {
public:
    int operator()(int a, int b) const { return a + b; }
};

int main(void) {
    std::function<int(int, int)> f;

    f = plain_add;                          // เก็บ function pointer ธรรมดา
    std::printf("%d\n", f(2, 3));

    f = Adder();                            // เก็บ functor
    std::printf("%d\n", f(2, 3));

    f = [](int a, int b) { return a * b; }; // เก็บ lambda
    std::printf("%d\n", f(2, 3));

    int captured = 100;
    f = [captured](int a, int b) { return a + b + captured; }; // เก็บ lambda ที่ capture ตัวแปร
    std::printf("%d\n", f(2, 3));

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 std_function_demo.cpp -o std_function_demo
./std_function_demo
```

ผลลัพธ์:

```
5
5
6
105
```

`std::function<int(int, int)>` อ่านว่า "ตัวแปรที่เก็บ Callable ใดๆ ที่รับ
พารามิเตอร์ `int` สองตัว และคืนค่าเป็น `int`" — ไม่ว่าตัวแปร `f` จะถูก assign
ด้วย function pointer, functor object, หรือ lambda expression (ทั้งแบบ
capture และไม่ capture) ก็เก็บได้หมด และเรียกใช้ด้วย `f(2, 3)` เหมือนกันทุก
ประการ นี่คือความสามารถพิเศษของ `std::function` ที่ Lambda เดี่ยวๆ หรือ
Function Template ทำไม่ได้ (เพราะ Template ต้อง fix ชนิดข้อมูลตอน compile-time
แต่ `std::function` เปลี่ยนสิ่งที่เก็บอยู่ข้างในได้ที่ runtime)

**ข้อควรรู้เรื่องประสิทธิภาพ**: `std::function` มี overhead มากกว่าการเรียก
Lambda หรือ Functor ตรงๆ เล็กน้อย (มักต้องมีการจัดสรรหน่วยความจำแบบ dynamic
เพิ่มเติมสำหรับ Callable ที่มีขนาดใหญ่ และมีการเรียกผ่าน virtual dispatch
ภายใน) ในโค้ดที่ต้องการประสิทธิภาพสูงสุด (เช่น loop ที่ทำงานหลายล้านรอบ) ควร
พิจารณาใช้ Template parameter รับ Callable ตรงๆ แทน (`template <typename F>
void call(F f)`) แต่ในโค้ดทั่วไป โดยเฉพาะระบบ Callback ที่ต้อง**เก็บ**
Callable ไว้ใช้ภายหลัง `std::function` คือทางเลือกที่เหมาะสมที่สุดเพราะความ
ยืดหยุ่นที่ได้มาคุ้มค่ากับ overhead เล็กน้อยที่เสียไป

---

## 64.8 ตัวอย่างจริง: ระบบ Callback ด้วย std::function (Step 512)

ตัวอย่างที่ใช้ `std::function` บ่อยที่สุดในโค้ดจริงคือ **ระบบ Callback** — เช่น
UI framework ที่ต้องการให้ผู้ใช้กำหนดว่า "เมื่อกดปุ่มนี้แล้วให้ทำอะไร" โดยไม่รู้
ล่วงหน้าว่าผู้ใช้จะเขียน logic แบบไหน:

```cpp
#include <cstdio>
#include <functional>
#include <string>
#include <utility>

class Button {
public:
    explicit Button(std::string label) : label_(std::move(label)) {}

    void set_on_click(std::function<void()> callback) {
        on_click_ = std::move(callback);
    }

    void click() const {
        std::printf("[%s] ถูกกด -> ", label_.c_str());
        if (on_click_) {
            on_click_();
        } else {
            std::printf("(ยังไม่ได้ตั้งค่า callback)\n");
        }
    }

private:
    std::string label_;
    std::function<void()> on_click_;
};

int main(void) {
    Button save_button("Save");
    Button cancel_button("Cancel");

    int save_count = 0;

    save_button.set_on_click([&save_count]() {
        ++save_count;
        std::printf("บันทึกข้อมูลแล้ว (ครั้งที่ %d)\n", save_count);
    });

    cancel_button.set_on_click([]() {
        std::printf("ยกเลิกการทำงาน\n");
    });

    save_button.click();
    save_button.click();
    cancel_button.click();

    Button unbound_button("Unbound");
    unbound_button.click();

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 callback_system.cpp -o callback_system
./callback_system
```

ผลลัพธ์:

```
[Save] ถูกกด -> บันทึกข้อมูลแล้ว (ครั้งที่ 1)
[Save] ถูกกด -> บันทึกข้อมูลแล้ว (ครั้งที่ 2)
[Cancel] ถูกกด -> ยกเลิกการทำงาน
[Unbound] ถูกกด -> (ยังไม่ได้ตั้งค่า callback)
```

class `Button` เก็บ `std::function<void()> on_click_` ไว้เป็น member — ตอน
สร้าง `Button` ยังไม่รู้เลยว่าเมื่อคลิกแล้วจะเกิดอะไรขึ้น (decoupling: แยกส่วน
"ปุ่ม" ออกจากส่วน "การกระทำเมื่อกด" อย่างสิ้นเชิง) ผู้ใช้ class `Button` เป็น
คนกำหนดพฤติกรรมทีหลังผ่าน `set_on_click()` ด้วย lambda expression ที่ capture
ตัวแปรภายนอกมาใช้ได้ตามต้องการ (เช่น `save_count` ที่ `save_button` capture by
reference มาเพื่อนับจำนวนครั้งที่กด) และ `if (on_click_)` ใช้ตรวจสอบว่า
`std::function` ตัวนั้นถูกกำหนดค่าไว้แล้วหรือยัง (ค่า default ของ
`std::function` ที่ยังไม่ได้ assign อะไรเลยจะแปลงเป็น `bool` ได้เป็น `false`)
ป้องกันการเรียก Callable ที่ว่างเปล่า (ซึ่งจะโยน `std::bad_function_call`
exception ถ้าเรียกโดยไม่เช็คก่อน)

รูปแบบนี้คือหัวใจของการออกแบบระบบ **Event-Driven** และ **Observer Pattern**
(จะพูดถึงอย่างเป็นทางการใน Part 98 เรื่อง Design Pattern เชิง Behavioral) ที่
ใช้กันอย่างแพร่หลายในโค้ดจริง ตั้งแต่ GUI Framework, Game Engine, ไปจนถึง
Networking Library ที่ต้อง "แจ้งเตือน" เมื่อมีเหตุการณ์เกิดขึ้น โดยไม่ต้องผูก
โค้ดส่วนที่ตรวจจับเหตุการณ์เข้ากับโค้ดส่วนที่ตอบสนองต่อเหตุการณ์นั้นโดยตรง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **Capture by reference แล้วปล่อยให้ lambda มีอายุยืนกว่าตัวแปรที่ capture
   มา (Dangling Reference)** — นี่คือบั๊กที่อันตรายที่สุดของหัวข้อนี้ เพราะ
   compile ผ่านและมักไม่มี warning ให้เห็นด้วย:

   ```cpp
   #include <cstdio>
   #include <functional>

   // !! ตัวอย่างบั๊กที่ตั้งใจสาธิต ห้ามเขียนแบบนี้ในโค้ดจริงเด็ดขาด !!
   std::function<int()> make_dangling_counter() {
       int local = 42;
       // อันตราย: capture by reference ตัวแปร local ที่เป็น stack variable ของฟังก์ชันนี้
       auto lam = [&local]() { return local; };
       return lam; // local จะถูกทำลายทันทีที่ฟังก์ชันนี้ return กลับไป
   }

   int main(void) {
       auto counter = make_dangling_counter();
       std::printf("%d\n", counter()); // Undefined Behavior: อ่านค่าจาก stack frame ที่ตายไปแล้ว
       return 0;
   }
   ```

   เมื่อ `make_dangling_counter()` return กลับไป ตัวแปร `local` (เป็น stack
   variable ของฟังก์ชันนั้น) จะถูกทำลายทันที แต่ lambda ที่ `main()` ได้รับมา
   ยังถือ **reference** ไปยังหน่วยความจำตำแหน่งเดิมอยู่ — การเรียก `counter()`
   ใน `main()` จึงเป็นการอ่านค่าจากหน่วยความจำที่ไม่มีความหมายอีกต่อไปแล้ว
   (Undefined Behavior) ในทางปฏิบัติโปรแกรมอาจพิมพ์ค่าขยะออกมา อาจดูเหมือน
   "ทำงานถูกต้องบังเอิญ" ในบางครั้ง หรืออาจ crash ก็ได้ ขึ้นกับสภาพหน่วยความจำ
   ขณะนั้น **วิธีป้องกัน**: ถ้า lambda จะถูกส่งออกไปใช้งานนอก scope ปัจจุบัน
   (เช่น เก็บใน `std::function` แล้วคืนค่าออกจากฟังก์ชัน หรือส่งไปเก็บใน
   container อื่น) ให้ **capture by value เสมอ** ไม่ใช่ by reference
2. **ใช้ `[=]` แบบเผลอ capture `this` ไปทั้ง object โดยไม่รู้ตัว** — ภายใน
   member function ถ้าเขียน `[=]` แล้วอ้างถึง member variable ของ class
   compiler จะ capture `this` pointer (ไม่ใช่คัดลอก object ทั้งก้อน) ซึ่งอาจ
   ทำให้เกิด dangling pointer ได้เช่นกันถ้า object ต้นทางถูกทำลายไปแล้วก่อนที่
   lambda จะถูกเรียก
3. **ลืมว่า `std::function` ที่ยังไม่ได้ assign ค่าใดๆ จะ throw exception เมื่อ
   ถูกเรียก** — ต้องเช็ค `if (callback)` ก่อนเรียกใช้เสมอ (ดังตัวอย่างใน 64.8)
   มิฉะนั้นจะได้ `std::bad_function_call` ที่โปรแกรม crash ทันทีถ้าไม่มีการ
   `catch` ไว้
4. **เขียน functor ที่มี state แล้วลืมว่า algorithm ของ STL ส่งต่อ functor
   แบบสำเนา (by value) เสมอ** — ถ้าไม่รับค่าที่ algorithm คืนกลับมา (เช่น
   `std::for_each`) state ที่สะสมไว้จะสูญหายไปกับสำเนาที่ algorithm ใช้ภายใน
5. **สับสนระหว่าง capture by value กับ mutable** — เข้าใจผิดว่าใส่ `mutable`
   แล้วจะแก้ไขตัวแปรต้นฉบับได้ ทั้งที่ `mutable` แค่อนุญาตให้แก้ไข **สำเนา**
   ภายใน lambda เท่านั้น ตัวแปรต้นฉบับนอก lambda ไม่เปลี่ยนแปลงเลย (ถ้าต้องการ
   แก้ไขตัวแปรต้นฉบับจริงๆ ต้องใช้ capture by reference แทน)

---

## แบบฝึกหัดท้ายบท

1. เขียน functor ชื่อ `Adder` ที่เก็บค่าคงที่ไว้ตอนสร้าง (ผ่าน constructor)
   แล้วใช้กับ `std::transform` เพื่อบวกค่าคงที่นั้นเข้ากับสมาชิกทุกตัวใน
   `std::vector<int>`
2. เขียน lambda expression ที่ capture ตัวแปร `threshold` แล้วใช้กับ
   `std::count_if` เพื่อนับจำนวนสมาชิกใน `std::vector<int>` ที่มีค่ามากกว่า
   `threshold`
3. เขียนโปรแกรมสาธิตปัญหา Dangling Reference จากการ capture by reference
   ตัวแปร local คล้ายตัวอย่างในหัวข้อ Common Pitfalls (ไม่จำเป็นต้องรันจริงก็
   ได้ แต่ต้อง compile ผ่าน) พร้อมอธิบายเป็นข้อความว่าทำไมโค้ดนี้ถึงอันตราย
4. สร้าง `std::vector<std::function<int(int)>>` เก็บฟังก์ชันหลายแบบ (บวกหนึ่ง,
   คูณสอง, ยกกำลังสอง) แล้ววนเรียกทีละตัวกับค่าตั้งต้นค่าหนึ่ง
5. เขียน generic lambda ที่รับ container ใดๆ ผ่าน `auto&` แล้วพิมพ์จำนวนสมาชิก
   ของมันด้วย `.size()`
6. อธิบายด้วยคำพูดของตัวเองว่า `mutable` lambda กับการ capture by reference
   ต่างกันอย่างไร ในแง่ของ "ใครเห็นการเปลี่ยนแปลงค่าบ้าง" หลังเรียก lambda จบ

### แนวทางเฉลยข้อ 1

```cpp
#include <cstdio>
#include <vector>
#include <algorithm>

class Adder {
public:
    explicit Adder(int amount) : amount_(amount) {}
    int operator()(int x) const { return x + amount_; }

private:
    int amount_;
};

int main(void) {
    std::vector<int> v = {1, 2, 3, 4, 5};
    std::vector<int> result(v.size());

    std::transform(v.begin(), v.end(), result.begin(), Adder(10));

    for (int x : result) std::printf("%d ", x);
    std::printf("\n");

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex1.cpp -o ex1
./ex1
# 11 12 13 14 15
```

`Adder(10)` สร้าง functor object ที่เก็บค่า `amount_ = 10` ไว้ แล้วส่งเข้า
`std::transform` เป็น Callable ตัวที่ 4 — `std::transform` จะเรียก
`operator()(x)` ของ `Adder` กับสมาชิกทุกตัวของ `v` แล้วเก็บผลลัพธ์ลงใน
`result` เทียบกับการเขียน lambda `[](int x) { return x + 10; }` แล้ว โค้ด
Functor ยาวกว่า แต่ถ้าต้องใช้ `Adder` แบบเดียวกันซ้ำในหลายจุดของโปรแกรม การ
ประกาศเป็น class ครั้งเดียวแล้วนำไปใช้ซ้ำจะดูแลรักษาง่ายกว่าการเขียน lambda
ซ้ำๆ ทุกจุด

### แนวทางเฉลยข้อ 4

```cpp
#include <cstdio>
#include <vector>
#include <functional>

int main(void) {
    std::vector<std::function<int(int)>> ops;

    ops.push_back([](int x) { return x + 1; });  // บวก
    ops.push_back([](int x) { return x * 2; });  // คูณ
    ops.push_back([](int x) { return x * x; });  // ยกกำลังสอง

    int value = 5;
    for (const auto& op : ops) {
        std::printf("%d -> %d\n", value, op(value));
    }

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex4.cpp -o ex4
./ex4
# 5 -> 6
# 5 -> 10
# 5 -> 25
```

เพราะ `std::vector` ต้องการให้สมาชิกทุกตัวเป็นชนิดข้อมูล**เดียวกัน**
(`std::vector<T>`) แต่ lambda สามแบบข้างต้นแต่ละตัวมี closure type ที่ต่างกัน
โดยธรรมชาติ (ตามที่อธิบายในหัวข้อ 64.7) จึงเก็บใน `std::vector<...>` ตรงๆ ไม่ได้
เลย ต้องห่อหุ้มด้วย `std::function<int(int)>` ก่อนเสมอ ซึ่งทำให้ทุก lambda
(หรือแม้แต่ functor/function pointer ที่มี signature ตรงกัน) กลายเป็นชนิด
ข้อมูลเดียวกันในสายตาของ `std::vector` ทำให้เก็บและวนลูปเรียกได้ในโค้ดเดียว —
นี่คือประโยชน์เชิงปฏิบัติที่ชัดเจนที่สุดของ `std::function`

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Function Object (Functor) คือ class ที่ overload `operator()`
  ทำให้ object เรียกใช้งานได้เหมือนฟังก์ชัน พร้อมความสามารถพิเศษในการเก็บ
  state ที่ฟังก์ชันธรรมดาทำไม่ได้
- เห็นเหตุผลว่าทำไม Functor ยังมีที่ยืนอยู่แม้ในยุคที่ Lambda ใช้งานสะดวกกว่า
  มากในกรณีทั่วไป
- เขียน Lambda Expression ในรูปแบบเต็ม `[capture](params) -> return_type
  { body }` ได้อย่างคล่องแคล่ว
- เข้าใจความแตกต่างและผลกระทบของ Capture by value กับ Capture by reference
  รวมถึงอันตรายของ Dangling Reference ที่ต้องระวังเป็นพิเศษ
- ใช้ `mutable` lambda และ Generic Lambda (C++14) ได้อย่างถูกต้อง
- ใช้ `std::function` เก็บ Callable ทุกชนิด (function pointer, functor,
  lambda) ไว้ในตัวแปรเดียวกันได้ และเข้าใจ trade-off ด้านประสิทธิภาพเมื่อ
  เทียบกับการเรียก Callable ตรงๆ
- ออกแบบระบบ Callback แบบง่ายด้วย `std::function` ซึ่งเป็นรากฐานของการออกแบบ
  ระบบ Event-Driven และ Observer Pattern ในโปรแกรมระดับ Production จริง

ตลอด Module E ที่ผ่านมา (Part 56-64) เราได้วางรากฐานเรื่อง Template, STL
Container, Iterator, Algorithm และ Callable object ครบถ้วนแล้ว ใน **Part 65**
เราจะกลับมาเจาะลึกเรื่อง **`std::string`** อีกครั้งในมุมที่ลึกกว่า Part 7
มาก รวมถึง `std::string_view` ที่ช่วยลดการคัดลอกข้อมูลโดยไม่จำเป็น และแนะนำ
การใช้ Regular Expression (`<regex>`) เบื้องต้นสำหรับการประมวลผลข้อความที่
ซับซ้อนขึ้น

**ต่อไป:** [Part 65 — std::string และ string_view/Regex](./part-065-string-advanced.md)
