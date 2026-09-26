# Part 65: std::string และ string_view/Regex ขั้นสูง (Step 513–520)

> Module E — Templates, Generic Programming และ STL | Part 65 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 513–520
> Part ก่อนหน้า: [Part 64 — Function Object, Lambda และ std::function](./part-064-lambda-functors.md) | Part ถัดไป: [Part 66 — Custom Allocator ใน STL](./part-066-custom-allocators.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างระหว่าง `std::string` กับ C-string (`char[]` / `char*`) จาก Part 7 ได้อย่าง
   ชัดเจน โดยเฉพาะเรื่องการจัดการหน่วยความจำอัตโนมัติ
2. ใช้เมธอดหลักของ `std::string` ได้อย่างคล่องแคล่ว: การต่อข้อความ (`operator+`, `+=`),
   `substr`, `find`, `replace` และการเปรียบเทียบ string
3. อธิบายแนวคิด Small String Optimization (SSO) และทำไมมันทำให้ string สั้นๆ เร็วขึ้นมาก
4. เข้าใจว่า `std::string_view` (C++17) คืออะไร ทำไมมันเร็วกว่า `std::string` ในหลายกรณี
   เพราะไม่มีการ copy ข้อมูล
5. ระบุและหลีกเลี่ยงอันตรายสำคัญที่สุดของ `string_view` คือ **dangling view** เมื่อข้อมูล
   ต้นทางถูกทำลายไปแล้วแต่ view ยังชี้ค้างอยู่
6. ใช้ `std::regex` เบื้องต้นได้จริง: `regex_match`, `regex_search`, `regex_replace` และ
   เขียน pattern สำหรับตรวจสอบรูปแบบอีเมลและเบอร์โทรศัพท์
7. เขียนฟังก์ชัน split/join ข้อความด้วยมือ และเข้าใจว่าทำไม C++ มาตรฐานถึงไม่มีฟังก์ชันนี้
   ให้ใช้ตรงๆ จนถึง C++20 (และแม้แต่ C++20 ก็ยังใช้งานอ้อมพอสมควร)

---

## 65.1 std::string เจาะลึก: เทียบกับ C-string (Step 513)

ใน **Part 7** เราเรียนเรื่อง C-string ไปแล้วว่ามันคือ `char` array ที่จบด้วย `'\0'`
(null terminator) ผู้เขียนโปรแกรมต้องรับผิดชอบเองทั้งหมดว่า:

- จะจองพื้นที่ (buffer) ขนาดเท่าไหร่ให้พอกับข้อความ
- ถ้าข้อความยาวขึ้นระหว่างทาง (เช่นต่อ string สองอันเข้าด้วยกัน) จะขยาย buffer อย่างไร
- ต้องเรียก `free()` เองถ้า buffer นั้นมาจาก `malloc`

`std::string` ใน C++ (อยู่ใน header `<string>`) ถูกออกแบบมาเพื่อแก้ปัญหาทั้งหมดนี้ โดยห่อ
(encapsulate) buffer ของตัวมันเองไว้ภายใน object แล้วจัดการ **จองหน่วยความจำ ขยาย
หน่วยความจำ และคืนหน่วยความจำ ให้อัตโนมัติทั้งหมด** ผ่านกลไก RAII (Resource Acquisition
Is Initialization ซึ่งจะเรียนเจาะลึกใน Part 68) — พูดง่ายๆ คือเราไม่ต้องคิดเรื่อง memory
management ของ string เองอีกต่อไป

```cpp
#include <cstring>
#include <iostream>
#include <string>

int main() {
    // C-string แบบเดิม (Part 7): ผู้เขียนโปรแกรมต้องคำนวณขนาด buffer เอง
    // "Hello" ยาว 5 ตัวอักษร + "World" ยาว 5 ตัวอักษร + '\0' = ต้องมีที่อย่างน้อย 11 ไบต์
    char c_greeting[11] = "Hello";
    std::strcat(c_greeting, "World");  // ถ้ากะขนาด buffer พลาดแม้แต่ 1 ไบต์ = buffer overflow ทันที

    std::cout << "C-string (ต้องคำนวณขนาดเอง): " << c_greeting << "\n";

    // std::string จัดการหน่วยความจำให้อัตโนมัติ ไม่ต้องกะขนาดล่วงหน้าเลย
    std::string cpp_greeting = "Hello";
    std::string cpp_name = "World";
    cpp_greeting += cpp_name;  // ขยายหน่วยความจำให้เองโดยอัตโนมัติ ไม่มีทางล้น buffer แบบข้างบน
    std::cout << "std::string (ปลอดภัย จัดการเอง): " << cpp_greeting << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 string_vs_cstring.cpp -o string_vs_cstring
./string_vs_cstring
```

ผลลัพธ์:

```
C-string (ต้องคำนวณขนาดเอง): HelloWorld
std::string (ปลอดภัย จัดการเอง): HelloWorld
```

สังเกตว่าโค้ดฝั่ง `std::string` ไม่มีตัวเลขขนาด buffer ปรากฏอยู่เลยแม้แต่ตัวเดียว เพราะ
`std::string` จะขยาย buffer ภายในให้เองทุกครั้งที่ข้อความยาวเกินความจุปัจจุบัน (`capacity()`)
โดยจะจอง heap memory ก้อนใหม่ที่ใหญ่กว่าเดิม copy ข้อมูลเก่าไปไว้ในก้อนใหม่ แล้วคืนก้อนเก่า
กลับให้ระบบอัตโนมัติ — ทั้งหมดนี้เกิดขึ้น "หลังฉาก" โดยที่เราไม่ต้องเขียนโค้ดจัดการเอง

### ตารางเปรียบเทียบ C-string กับ std::string

| ประเด็น | C-string (`char[]` / `char*`) | `std::string` |
|---|---|---|
| การจัดการหน่วยความจำ | ผู้เขียนโปรแกรมทำเอง (`malloc`/`free`) | อัตโนมัติผ่าน RAII |
| ทราบความยาวได้อย่างไร | ต้องวิ่งหา `'\0'` (`strlen`, O(n)) | เก็บความยาวไว้ในตัว เรียก `.size()` O(1) |
| ต่อข้อความ (concatenation) | `strcat`/`strncat` ต้องกะขนาด buffer เอง | `operator+`, `+=` ขยายให้อัตโนมัติ |
| เปรียบเทียบ | `strcmp` (คืนค่า int ต้องแปลความหมาย) | `operator==`, `<`, `>` ใช้ตรงๆ |
| ความปลอดภัยจาก buffer overflow | ไม่มีการป้องกันเลย ขึ้นกับผู้เขียนโปรแกรม | ปลอดภัยกว่ามากเพราะขยายอัตโนมัติ |
| คัดลอก (copy) | ต้อง `strcpy`/`strncpy` เอง | `operator=` ทำให้อัตโนมัติ (deep copy) |
| ใช้กับ STL container/algorithm | ใช้ยาก ต้องเขียน custom comparator | ใช้ร่วมกับ container/algorithm ได้ตรงๆ |

> **หมายเหตุสำคัญ**: `std::string` ไม่ได้ "แทนที่" C-string ไปเสียทีเดียว เพราะ C API จำนวนมาก
> (เช่นฟังก์ชันของระบบปฏิบัติการ, C library เก่าๆ) ยังคงรับ `const char*` อยู่
> `std::string` มีเมธอด `.c_str()` เพื่อดึง C-string แบบ null-terminated ออกมาใช้กับ API เหล่านั้น
> เสมอเมื่อจำเป็น

---

## 65.2 การจัดการข้อความด้วย std::string (Step 514)

`std::string` มีเมธอดให้ใช้งานเยอะมาก แต่ที่ใช้บ่อยที่สุดในงานจริงมีอยู่ไม่กี่ตัว:
การต่อข้อความ, การตัดข้อความย่อย (`substr`), การค้นหา (`find`) และการแทนที่ (`replace`)

```cpp
#include <iostream>
#include <string>

int main() {
    std::string sentence = "The quick brown fox jumps over the lazy dog";

    // 1) operator+ / operator+= สำหรับต่อ string
    std::string greeting = "Hello, " + std::string("Somchai") + "!";
    std::cout << "greeting: " << greeting << "\n";

    // 2) substr(pos, len) — ดึงข้อความบางส่วนออกมาเป็น string ใหม่
    std::string word = sentence.substr(4, 5);  // เริ่มที่ index 4 ยาว 5 ตัวอักษร
    std::cout << "substr(4, 5): " << word << "\n";

    // 3) find — หาตำแหน่งของ substring, คืนค่า std::string::npos ถ้าไม่เจอ
    std::size_t pos = sentence.find("fox");
    if (pos != std::string::npos) {
        std::cout << "พบคำว่า \"fox\" ที่ตำแหน่ง: " << pos << "\n";
    }

    std::size_t not_found = sentence.find("cat");
    std::cout << "หาคำว่า \"cat\" เจอไหม: "
              << (not_found == std::string::npos ? "ไม่เจอ" : "เจอ") << "\n";

    // 4) replace — แทนที่ข้อความช่วงหนึ่งด้วยข้อความใหม่
    std::string edited = sentence;
    edited.replace(16, 3, "cat");  // แทนที่ "fox" (เริ่มที่ 16, ยาว 3) ด้วย "cat"
    std::cout << "หลัง replace: " << edited << "\n";

    // 5) การเปรียบเทียบ string ทำได้ตรงๆ ด้วย operator== ต่างจาก C ที่ต้องใช้ strcmp
    std::string a = "apple";
    std::string b = "apple";
    std::cout << "a == b: " << std::boolalpha << (a == b) << "\n";

    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 string_ops.cpp -o string_ops && ./string_ops
```

ผลลัพธ์:

```
greeting: Hello, Somchai!
substr(4, 5): quick
พบคำว่า "fox" ที่ตำแหน่ง: 16
หาคำว่า "cat" เจอไหม: ไม่เจอ
หลัง replace: The quick brown cat jumps over the lazy dog
a == b: true
```

### จุดที่ต้องระวังเรื่อง `operator+`

```cpp
#include <iostream>
#include <string>

int main() {
    // ผิด: "Hello, " เป็น const char* ต่อกับ "Somchai" ซึ่งเป็น const char* อีกตัว
    // C++ ไม่อนุญาตให้ operator+ ทำงานกับ const char* สองตัวพร้อมกัน (compile error)
    // std::string result = "Hello, " + "Somchai";

    // ถูก: ต้องมี std::string อย่างน้อยหนึ่งฝั่งเสมอ เพื่อให้ operator+ ของ std::string ทำงานได้
    std::string result1 = std::string("Hello, ") + "Somchai";  // ถูก
    std::string result2 = "Hello, " + std::string("Somchai");  // ถูก

    std::cout << result1 << "\n" << result2 << "\n";

    return 0;
}
```

นี่คือความเข้าใจผิดที่พบบ่อยมากสำหรับผู้เริ่มต้น: `operator+` ที่ต่อ string เข้าด้วยกันได้
เป็น operator ของ `std::string` ไม่ใช่ของ `const char*` ดังนั้นต้องมี `std::string`
อยู่อย่างน้อยหนึ่งฝั่งของ `+` เสมอ compiler จึงจะรู้ว่าต้องเรียก operator ตัวไหน

### เมธอดที่ควรรู้จักเพิ่มเติม

| เมธอด | หน้าที่ |
|---|---|
| `.length()` / `.size()` | คืนจำนวนตัวอักษร (ทั้งสองตัวทำงานเหมือนกันทุกประการ) |
| `.empty()` | คืน `true` ถ้า string ว่าง (เร็วกว่าเช็ค `.size() == 0`) |
| `.append(...)` | ต่อข้อความ เทียบเท่ากับ `+=` แต่ยืดหยุ่นกว่า (รับ iterator range ได้) |
| `.insert(pos, str)` | แทรกข้อความเข้าไปที่ตำแหน่งที่กำหนด |
| `.erase(pos, len)` | ลบข้อความช่วงหนึ่งออก |
| `.compare(other)` | เปรียบเทียบแบบ lexicographic คืนค่า int (เหมือน `strcmp`) |
| `.at(pos)` | เข้าถึงตัวอักษร พร้อมตรวจสอบขอบเขต (throw `std::out_of_range` ถ้าเกิน) |
| `operator[]` | เข้าถึงตัวอักษร **ไม่ตรวจสอบขอบเขต** (เร็วกว่าแต่เสี่ยง Undefined Behavior) |

---

## 65.3 Small String Optimization (SSO) (Step 515)

ถ้า `std::string` ต้องขอ heap memory ทุกครั้งที่สร้าง object ใหม่ การเขียนโปรแกรมที่สร้าง
string สั้นๆ จำนวนมาก (เช่น `"OK"`, `"true"`, ตัวแปรชื่อคน) จะช้ามาก เพราะการขอ/คืน heap
memory (ผ่าน `malloc`/`free`) มีค่าใช้จ่าย (overhead) สูงกว่าการใช้ stack memory มาก

Compiler ยุคใหม่แทบทุกตัว (GCC/libstdc++, Clang/libc++, MSVC) จึงimplement เทคนิคที่เรียกว่า
**Small String Optimization (SSO)**: ถ้า string สั้นพอ (ขึ้นกับ implementation แต่ส่วนใหญ่
อยู่ที่ประมาณ 15-22 ตัวอักษร) `std::string` จะเก็บข้อมูลไว้ **ภายใน object เอง** (บน stack)
แทนที่จะไปขอ heap memory เลย ทำให้ string สั้นๆ เร็วขึ้นมากและไม่มี heap allocation เลย

```cpp
#include <iostream>
#include <string>

// ฟังก์ชันช่วยเช็คว่า string ปัจจุบันใช้ Small String Optimization (SSO) อยู่หรือไม่
// เทคนิค: ถ้าที่อยู่ของ buffer ข้อมูล (data()) อยู่ "ภายใน" ขอบเขตของ object เอง
// แปลว่ามันเก็บข้อมูลอยู่บน stack ของ object นั้น (SSO) ไม่ได้ไปขอ heap
bool uses_sso(const std::string& s) {
    const void* obj_start = &s;
    const void* obj_end = reinterpret_cast<const char*>(&s) + sizeof(std::string);
    const void* data_ptr = s.data();
    return data_ptr >= obj_start && data_ptr < obj_end;
}

int main() {
    std::string short_str = "Hi";                              // สั้นมาก
    std::string medium_str = "Hello";                          // ยังสั้น
    std::string long_str = "This string is definitely long enough to exceed SSO buffer";

    std::cout << "sizeof(std::string) = " << sizeof(std::string) << " bytes\n\n";

    std::cout << "\"" << short_str << "\" (length " << short_str.size()
              << ") ใช้ SSO: " << std::boolalpha << uses_sso(short_str) << "\n";
    std::cout << "\"" << medium_str << "\" (length " << medium_str.size()
              << ") ใช้ SSO: " << uses_sso(medium_str) << "\n";
    std::cout << "\"" << long_str.substr(0, 20) << "...\" (length " << long_str.size()
              << ") ใช้ SSO: " << uses_sso(long_str) << "\n";

    return 0;
}
```

ผลลัพธ์ (ทดสอบด้วย GCC 13 / libstdc++ บน Linux x86-64):

```
sizeof(std::string) = 32 bytes

"Hi" (length 2) ใช้ SSO: true
"Hello" (length 5) ใช้ SSO: true
"This string is defin..." (length 58) ใช้ SSO: false
```

`libstdc++` ของ GCC เก็บ SSO buffer ได้สูงสุด 15 ตัวอักษร (บวก null terminator รวมเป็น 16
ไบต์) ภายใน object ขนาด 32 ไบต์ ส่วน `libc++` ของ Clang ใช้ตัวเลขที่ต่างออกไปเล็กน้อย
(ประมาณ 22 ตัวอักษร) — **นี่คือรายละเอียดที่ implementation-defined** มาตรฐาน C++ ไม่ได้
บังคับว่าต้องมี SSO เลยด้วยซ้ำ แต่ในทางปฏิบัติทุก compiler หลักที่ใช้งานจริงในปี 2026
ต่างก็ implement เทคนิคนี้ทั้งสิ้น เพราะประโยชน์ด้านประสิทธิภาพชัดเจนมาก

### ทำไม SSO ถึงสำคัญในทางปฏิบัติ

- ตัวแปร string สั้นๆ ที่ใช้เป็น key ใน map, ชื่อ field, สถานะ (`"active"`, `"pending"`) ฯลฯ
  จะไม่ทำให้เกิด heap allocation เลยแม้แต่ครั้งเดียวตลอดอายุของโปรแกรม
- การ copy string สั้นๆ (เช่นตอนส่งเป็นพารามิเตอร์แบบ pass-by-value) เร็วกว่าการ copy string
  ยาวมาก เพราะแค่ copy ข้อมูลบน stack ไม่ต้องขอ heap memory ใหม่
- นี่คือเหตุผลหนึ่งที่ทำให้คำแนะนำ "อย่า `substr`/`+=` string สั้นๆ พร่ำเพรื่อเพราะกลัว
  performance" ไม่ค่อยจำเป็นเท่าที่คิด ตราบใดที่ string ยังอยู่ในขนาด SSO

> **ข้อควรระวัง**: อย่าเขียนโค้ด production ที่พึ่งพาตัวเลข "15 ตัวอักษร" หรือ layout ภายใน
> ของ `std::string` แบบนี้ตรงๆ (เทคนิค `uses_sso` ข้างต้นเป็นแค่การสาธิตเพื่อการศึกษา
> เพราะ layout ภายในของ `std::string` เป็นรายละเอียด implementation-defined ที่อาจ
> เปลี่ยนแปลงได้ระหว่าง compiler หรือแม้แต่ระหว่างเวอร์ชันของ compiler เดียวกัน)

---

## 65.4 std::string_view คืออะไร และทำไมเร็วกว่า (Step 516)

ปัญหาของ `std::string` คือมันเป็น **owning type** — ทุกครั้งที่สร้าง `std::string` ใหม่จาก
ข้อมูลเดิม (เช่นรับพารามิเตอร์เป็น `const std::string&` จาก string literal) หรือใช้
`.substr()` (ก่อน C++17 string_view) จะเกิดการ **copy ข้อมูล** เสมอ ถ้าฟังก์ชันแค่ต้องการ
"อ่าน" ข้อมูลโดยไม่ได้ต้องการเป็นเจ้าของมัน การ copy นี้เป็นค่าใช้จ่ายที่ไม่จำเป็นเลย

C++17 จึงเพิ่ม `std::string_view` (อยู่ใน header `<string_view>`) เข้ามา — มันคือ
**non-owning view**: object ที่เก็บแค่ **pointer ไปยังข้อมูล + ความยาว** (คู่ `(ptr, len)`)
ไม่ได้เป็นเจ้าของหน่วยความจำใดๆ เลย จึงไม่มีการจัดสรร (allocate) หรือ copy ข้อมูลเกิดขึ้น
เมื่อสร้าง `string_view` จาก string ที่มีอยู่แล้ว (ไม่ว่าจะเป็น `std::string`, string literal
หรือ `char*`)

```cpp
#include <iostream>
#include <string>
#include <string_view>

// รับเป็น const std::string& — ถ้าผู้เรียกส่ง const char* หรือ string literal เข้ามา
// จะต้อง "สร้าง" std::string ชั่วคราวขึ้นมาก่อน (มีการจัดสรรหน่วยความจำ / copy ข้อมูล)
std::size_t count_vowels_by_string(const std::string& s) {
    std::size_t count = 0;
    for (char c : s) {
        if (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') {
            ++count;
        }
    }
    return count;
}

// รับเป็น std::string_view — เป็นแค่ "คู่ (pointer, length)" ที่ชี้ไปยังข้อมูลเดิม
// ไม่ว่าผู้เรียกจะส่ง const char*, string literal หรือ std::string เข้ามา
// ก็ไม่มีการ copy ข้อมูลเกิดขึ้นเลยแม้แต่ไบต์เดียว
std::size_t count_vowels_by_view(std::string_view s) {
    std::size_t count = 0;
    for (char c : s) {
        if (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') {
            ++count;
        }
    }
    return count;
}

int main() {
    // string_view สร้างจาก string literal โดยตรง ไม่มีการจัดสรรหน่วยความจำเลย
    std::string_view view_literal = "Hello, World!";
    std::cout << "view_literal: " << view_literal << " (size=" << view_literal.size() << ")\n";

    // string_view ยังใช้ substr ได้เหมือน string แต่ไม่ copy ข้อมูล แค่ขยับ pointer/length
    std::string_view sub = view_literal.substr(7, 5);
    std::cout << "substr(7, 5) แบบ view: " << sub << "\n";

    std::string owner = "The quick brown fox jumps over the lazy dog";

    // เรียกทั้งสองแบบด้วยข้อมูลเดียวกัน ผลลัพธ์ต้องเท่ากัน
    std::cout << "count_vowels_by_string: " << count_vowels_by_string(owner) << "\n";
    std::cout << "count_vowels_by_view:   " << count_vowels_by_view(owner) << "\n";

    // จุดสำคัญ: ฟังก์ชันที่รับ string_view เรียกด้วย literal ได้โดยไม่สร้าง std::string ชั่วคราวเลย
    std::cout << "count_vowels_by_view(\"literal\"): "
              << count_vowels_by_view("this is a raw literal, no std::string created") << "\n";

    return 0;
}
```

ผลลัพธ์:

```
view_literal: Hello, World! (size=13)
substr(7, 5) แบบ view: World
count_vowels_by_string: 11
count_vowels_by_view:   11
count_vowels_by_view("literal"): 12
```

### ทำไม `count_vowels_by_string` ถึงช้ากว่าในบางกรณี

ถ้าเรียก `count_vowels_by_string("some literal")` ด้วย string literal ตรงๆ compiler จะต้อง
**สร้าง `std::string` ชั่วคราว** ขึ้นมาก่อน (implicit conversion จาก `const char*` เป็น
`std::string`) ซึ่งถ้า literal นั้นยาวเกิน SSO buffer จะมี heap allocation เกิดขึ้นทันที
แล้วพอฟังก์ชันจบก็ต้องคืนหน่วยความจำนั้นทิ้งไปทันที — เสียเวลาโดยเปล่าประโยชน์เพราะฟังก์ชัน
แค่ "อ่าน" ข้อมูลเท่านั้น ไม่ได้ต้องการเป็นเจ้าของมันเลย ในขณะที่ `count_vowels_by_view`
รับ `string_view` เข้ามาโดยตรงโดยไม่มี allocation ใดๆ เกิดขึ้นเลย

### กฎทองในการเลือกใช้พารามิเตอร์

| สถานการณ์ | ควรใช้ |
|---|---|
| ฟังก์ชันแค่ "อ่าน" ข้อความ ไม่เก็บไว้ใช้ภายหลัง | `std::string_view` (พารามิเตอร์ pass-by-value) |
| ฟังก์ชันต้อง "เก็บ" ข้อความไว้ใช้ต่อ (เช่นเป็น field ของ class) | `std::string` (รับมาแล้ว copy/move เก็บไว้) |
| ฟังก์ชันต้องแก้ไขข้อความ (เช่น `to_upper` แบบ in-place) | `std::string&` |
| คืนค่า string ที่สร้างใหม่ในฟังก์ชัน | คืนเป็น `std::string` (ห้ามคืนเป็น `string_view`! ดู 65.5) |

---

## 65.5 อันตรายของ string_view: Dangling View (Step 517)

เพราะ `std::string_view` **ไม่ได้เป็นเจ้าของข้อมูล** มันจึงมีสมมติฐานสำคัญข้อหนึ่งที่ผู้เขียน
โปรแกรมต้องรับผิดชอบเอง: **ข้อมูลต้นทางที่ view ชี้ไปต้องยังมีชีวิตอยู่ตลอดเวลาที่ view
ถูกใช้งาน** ถ้าข้อมูลต้นทางถูกทำลายไปแล้วแต่ view ยังชี้ค้างอยู่ — เรียกว่า **dangling
string_view** — การใช้งาน view นั้นต่อไปคือ **Undefined Behavior (UB)** ทันที

```cpp
// *** ตัวอย่างนี้จงใจสาธิตบั๊ก (dangling string_view) ***
// โปรแกรมนี้คอมไพล์ผ่านโดยไม่มี warning แต่มี Undefined Behavior ตอนรัน
// ห้ามเขียนโค้ดแบบนี้ในงานจริงเด็ดขาด — ดูคำอธิบายด้านล่าง
#include <iostream>
#include <string>
#include <string_view>

// อันตราย: ฟังก์ชันนี้สร้าง std::string ชั่วคราวไว้ในตัวแปร local
// แล้วคืนค่าเป็น string_view ที่ "ชี้" ไปยังข้อมูลของ local นั้น
std::string_view make_dangling_view() {
    std::string local = "ข้อมูลนี้จะหายไปเมื่อฟังก์ชันจบ";
    return local;  // local ถูกทำลายทันทีที่ฟังก์ชัน return กลับไป
}                  // แต่ string_view ที่ส่งออกไปยังชี้ไปยังหน่วยความจำที่ถูกคืนแล้ว

int main() {
    std::string_view dangling = make_dangling_view();

    // ต่อจากนี้คือ Undefined Behavior: dangling ชี้ไปยังหน่วยความจำที่ถูกปล่อยคืนไปแล้ว
    // ผลลัพธ์ที่พิมพ์ออกมาอาจเป็นขยะ อาจ crash หรือบังเอิญยังถูกอยู่ก็ได้ — คาดเดาไม่ได้เลย
    std::cout << "dangling view: " << dangling << "\n";

    return 0;
}
```

ผลลัพธ์จริงที่ได้ (จะแตกต่างกันไปในแต่ละครั้งที่รัน แต่ในเครื่องทดสอบนี้ได้ขยะปนมา):

```
dangling view: B�X   �g�,�1@��นี้จะหายไปเมื่อฟังก์ชันจบ
```

สังเกตว่า **compiler ไม่เตือนอะไรเลยแม้เปิด `-Wall -Wextra -Wpedantic`** เพราะในทาง
ไวยากรณ์โค้ดนี้ถูกต้องสมบูรณ์แบบ — ปัญหาอยู่ที่ "ตรรกะของอายุการใช้งาน (lifetime)" ซึ่ง
compiler ทั่วไปตรวจจับไม่ได้ (compiler รุ่นใหม่ๆ ที่มี `-Wdangling-pointer` หรือ static
analyzer อย่าง clang-tidy อาจจับกรณีง่ายๆ แบบนี้ได้บ้าง แต่ไม่ครอบคลุมทุกกรณี)

### สาเหตุที่พบบ่อยของ dangling string_view

1. **คืนค่า `string_view` จากฟังก์ชันที่สร้าง `std::string` local** (ตามตัวอย่างข้างบน)
   — วิธีแก้: คืนค่าเป็น `std::string` แทน (จะมี move semantics ทำให้ไม่เสีย performance
   มากอย่างที่คิด รายละเอียดใน Part 70)
2. **เก็บ `string_view` ไว้เป็น member ของ class ที่ชี้ไปยัง `std::string` ที่ถูกทำลายไปแล้ว**
   เช่น class เก็บ `string_view` ที่ชี้ไปยัง `std::string` ของ object อื่นที่ scope สั้นกว่า
3. **string_view ที่ชี้ไปยัง `std::string` ที่ถูก reallocate** — ถ้า `std::string` ต้นทาง
   ถูกแก้ไขจนต้องขยาย buffer (เช่นเรียก `+=` จนเกิน capacity เดิม) pointer เดิมที่ view
   ถืออยู่จะกลายเป็น dangling ทันที แม้ว่า `std::string` ตัวนั้นจะยังไม่ถูกทำลายก็ตาม

```cpp
#include <iostream>
#include <string>
#include <string_view>

int main() {
    std::string data = "short";
    std::string_view view = data;  // ตอนนี้ view ยังปลอดภัย ชี้ไปยัง buffer ของ data

    std::cout << "ก่อนแก้ไข: " << view << "\n";

    // เมื่อ data ถูกขยายจนต้อง reallocate buffer ใหม่ (เกินความจุ SSO/capacity เดิม)
    // pointer ภายในของ data จะเปลี่ยนไปชี้ buffer ก้อนใหม่ แต่ view ยังถือ pointer เก่าอยู่!
    data += " but now this string has become long enough to force reallocation";

    // *** ห้ามใช้ view ต่อจากจุดนี้ — เป็น Undefined Behavior เพราะ buffer เก่าถูกคืนไปแล้ว ***
    // std::cout << "หลังแก้ไข (อันตราย): " << view << "\n";  // จงใจ comment ไว้ ไม่รันบรรทัดนี้

    // วิธีที่ปลอดภัย: สร้าง view ใหม่จาก data หลังแก้ไขเสร็จแล้วเท่านั้น
    std::string_view fresh_view = data;
    std::cout << "view ใหม่หลังแก้ไข (ปลอดภัย): " << fresh_view << "\n";

    return 0;
}
```

### กฎการใช้งาน `string_view` อย่างปลอดภัย

- ใช้ `string_view` เป็น **พารามิเตอร์ของฟังก์ชัน** ที่แค่อ่านข้อมูลชั่วคราวเท่านั้น
  (lifetime สั้น ชัดเจน อยู่ในขอบเขตการเรียกฟังก์ชันเดียว)
- **ห้ามคืนค่า `string_view`** ที่ชี้ไปยังข้อมูลที่สร้างขึ้นภายในฟังก์ชันนั้นเอง
- **ห้ามเก็บ `string_view` ไว้ยาวนาน** (เช่นเป็น member ของ class ที่มีอายุยืนกว่าต้นทาง)
  เว้นแต่มั่นใจ 100% ว่าต้นทางจะมีชีวิตอยู่นานกว่า view เสมอ
- ถ้าต้นทางเป็น string ที่อาจถูกแก้ไข (mutate) หลังสร้าง view ไว้ ให้ระวังเรื่อง
  reallocation เสมอ — ปลอดภัยที่สุดคือสร้าง view ใหม่ทุกครั้งหลังข้อมูลถูกแก้ไข

---

## 65.6 std::regex เบื้องต้น: regex_match และ regex_search (Step 518)

Regular Expression (regex หรือ regex) คือภาษาเล็กๆ สำหรับอธิบาย "รูปแบบ" ของข้อความ
ใช้ตรวจสอบว่าข้อความตรงกับรูปแบบที่กำหนดหรือไม่ ค้นหา หรือแทนที่ข้อความที่ตรงรูปแบบ
C++ มี `std::regex` ให้ใช้ในตัวผ่าน header `<regex>` โดยรองรับ syntax แบบ ECMAScript
เป็นค่าเริ่มต้น (เหมือน regex ที่ใช้ใน JavaScript, กว้างขวางและคุ้นเคยที่สุด)

```cpp
#include <iostream>
#include <regex>
#include <string>

int main() {
    // regex_match: ตรวจว่า "ทั้ง string" ตรงกับ pattern พอดีทุกตัวอักษรหรือไม่
    std::regex digits_only(R"(\d+)");  // raw string literal R"(...)" กัน backslash ยุ่งยาก

    std::string a = "12345";
    std::string b = "12345abc";

    std::cout << "\"" << a << "\" ตรงกับ \\d+ ทั้ง string: "
              << std::boolalpha << std::regex_match(a, digits_only) << "\n";
    std::cout << "\"" << b << "\" ตรงกับ \\d+ ทั้ง string: "
              << std::regex_match(b, digits_only) << "\n";

    // regex_search: ตรวจว่ามี "บางส่วน" ของ string ที่ตรงกับ pattern หรือไม่
    std::cout << "\"" << b << "\" มีส่วนที่ตรงกับ \\d+ หรือไม่: "
              << std::regex_search(b, digits_only) << "\n";

    // ดึงข้อความที่ match ออกมาด้วย std::smatch
    std::smatch match_result;
    std::string log_line = "user_id=4821 requested at timestamp=1732000000";
    std::regex number_pattern(R"(\d+)");

    if (std::regex_search(log_line, match_result, number_pattern)) {
        std::cout << "เจอตัวเลขตัวแรก: " << match_result.str(0)
                  << " ที่ตำแหน่ง: " << match_result.position(0) << "\n";
    }

    // วนหาทุกตำแหน่งที่ match ด้วย sregex_iterator
    std::cout << "ตัวเลขทั้งหมดใน log_line: ";
    auto begin = std::sregex_iterator(log_line.begin(), log_line.end(), number_pattern);
    auto end = std::sregex_iterator();
    for (auto it = begin; it != end; ++it) {
        std::cout << it->str() << " ";
    }
    std::cout << "\n";

    return 0;
}
```

ผลลัพธ์:

```
"12345" ตรงกับ \d+ ทั้ง string: true
"12345abc" ตรงกับ \d+ ทั้ง string: false
"12345abc" มีส่วนที่ตรงกับ \d+ หรือไม่: true
เจอตัวเลขตัวแรก: 4821 ที่ตำแหน่ง: 8
ตัวเลขทั้งหมดใน log_line: 4821 1732000000
```

### ความแตกต่างระหว่าง `regex_match` และ `regex_search`

| ฟังก์ชัน | ความหมาย | ตัวอย่างการใช้งาน |
|---|---|---|
| `std::regex_match` | ตรวจว่า **ทั้ง string** ตรงกับ pattern พอดี (เหมือนใส่ `^...$` ล้อมรอบ) | validate format เช่น "ทั้ง string นี้เป็นอีเมลที่ถูกต้องหรือไม่" |
| `std::regex_search` | ตรวจว่า **มีบางส่วน** ของ string ตรงกับ pattern | ค้นหา/ดึงข้อมูลบางส่วนจาก string ที่ยาวกว่า pattern |
| `std::regex_replace` | แทนที่ทุกส่วนที่ตรง pattern ด้วยข้อความใหม่ | ปิดบังข้อมูลอ่อนไหว, format ข้อความใหม่ |

### สัญลักษณ์ regex พื้นฐานที่ใช้บ่อย

| สัญลักษณ์ | ความหมาย |
|---|---|
| `\d` | ตัวเลข 0-9 หนึ่งตัว |
| `\w` | ตัวอักษร, ตัวเลข หรือ `_` หนึ่งตัว |
| `\s` | ช่องว่าง (space, tab, newline) |
| `.` | ตัวอักษรอะไรก็ได้หนึ่งตัว |
| `+` | ตัวก่อนหน้าซ้ำ 1 ครั้งขึ้นไป |
| `*` | ตัวก่อนหน้าซ้ำ 0 ครั้งขึ้นไป |
| `?` | ตัวก่อนหน้ามีหรือไม่มีก็ได้ (0 หรือ 1 ครั้ง) |
| `{n,m}` | ตัวก่อนหน้าซ้ำระหว่าง n ถึง m ครั้ง |
| `[...]` | ตัวอักษรหนึ่งตัวจากกลุ่มที่ระบุ เช่น `[A-Za-z]` |
| `^` / `$` | จุดเริ่มต้น / จุดสิ้นสุดของ string |
| `(...)` | จัดกลุ่ม (capture group) สำหรับดึงข้อมูลออกมาทีหลัง |

> **หมายเหตุเรื่องประสิทธิภาพ**: การสร้าง `std::regex` object มีค่าใช้จ่ายสูงพอสมควร
> (ต้อง compile pattern เป็น state machine ภายใน) ถ้าต้องใช้ pattern เดิมซ้ำๆ ในลูป
> ควรสร้าง `std::regex` object ไว้ **ครั้งเดียวนอกลูป** (หรือประกาศเป็น `static const`
> ถ้าอยู่ในฟังก์ชันที่เรียกซ้ำ) ไม่ใช่สร้างใหม่ทุกรอบ

---

## 65.7 regex_replace และตัวอย่างจริง: Validate Email (Step 519)

หนึ่งในการใช้งาน regex ที่พบบ่อยที่สุดในงานจริงคือการตรวจสอบรูปแบบข้อมูลที่ผู้ใช้กรอก เช่น
อีเมล เบอร์โทรศัพท์ หรือรหัสไปรษณีย์ ก่อนนำไปประมวลผลต่อ

```cpp
#include <iostream>
#include <regex>
#include <string>
#include <vector>

// pattern เบื้องต้นสำหรับตรวจรูปแบบอีเมล (ไม่ครอบคลุมมาตรฐาน RFC 5322 เต็มรูปแบบ
// ซึ่งซับซ้อนมาก แต่เพียงพอสำหรับกรอง format ผิดพลาดทั่วไปในฟอร์มสมัครสมาชิก)
bool is_valid_email(const std::string& email) {
    static const std::regex email_pattern(
        R"(^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$)");
    return std::regex_match(email, email_pattern);
}

int main() {
    std::vector<std::string> candidates = {
        "somchai@example.com",
        "prasit.k@company.co.th",
        "invalid-email",
        "missing@dot",
        "@nodomain.com",
        "name@sub.domain.io",
    };

    for (const auto& email : candidates) {
        std::cout << email << " -> "
                  << (is_valid_email(email) ? "ถูกต้อง" : "ไม่ถูกต้อง") << "\n";
    }

    // regex_replace: แทนที่ทุกตำแหน่งที่ match ด้วยข้อความใหม่
    std::string message = "ติดต่อ somchai@example.com หรือ prasit.k@company.co.th ได้เลย";
    std::regex any_email(R"([A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,})");
    std::string masked = std::regex_replace(message, any_email, "[EMAIL HIDDEN]");
    std::cout << "\nข้อความต้นฉบับ: " << message << "\n";
    std::cout << "หลังปกปิดอีเมล: " << masked << "\n";

    return 0;
}
```

ผลลัพธ์:

```
somchai@example.com -> ถูกต้อง
prasit.k@company.co.th -> ถูกต้อง
invalid-email -> ไม่ถูกต้อง
missing@dot -> ไม่ถูกต้อง
@nodomain.com -> ไม่ถูกต้อง
name@sub.domain.io -> ถูกต้อง

ข้อความต้นฉบับ: ติดต่อ somchai@example.com หรือ prasit.k@company.co.th ได้เลย
หลังปกปิดอีเมล: ติดต่อ [EMAIL HIDDEN] หรือ [EMAIL HIDDEN] ได้เลย
```

สังเกตการใช้ `static const std::regex` ภายในฟังก์ชัน `is_valid_email` — เพราะฟังก์ชันนี้
อาจถูกเรียกซ้ำหลายพันครั้ง (เช่นตรวจสอบทุกแถวจากไฟล์ CSV ที่มีผู้ใช้เป็นหมื่นคน) การใช้
`static const` ทำให้ pattern ถูก compile แค่ครั้งเดียวตลอดอายุโปรแกรม ไม่ใช่ทุกครั้งที่
เรียกฟังก์ชัน

> **ข้อควรระวังสำคัญ**: pattern อีเมลข้างบนนี้เป็นการตรวจสอบแบบ "เพียงพอสำหรับงานทั่วไป"
> เท่านั้น ไม่ใช่การตรวจสอบตามมาตรฐาน RFC 5322 แบบสมบูรณ์ (ซึ่งซับซ้อนมากจนแทบไม่มีใครเขียน
> regex ให้ครอบคลุม 100% จริงๆ) ในงาน production ที่ต้องการความแม่นยำสูง (เช่นระบบสมัคร
> สมาชิก) ควรใช้ pattern ตรวจสอบ format พื้นฐานแบบนี้ร่วมกับการส่งอีเมลยืนยัน (verification
> email) เพื่อพิสูจน์ว่าอีเมลนั้นมีอยู่จริงและผู้ใช้เป็นเจ้าของจริง

---

## 65.8 String Splitting และ Joining ด้วยมือ (Step 520)

หนึ่งในสิ่งที่ทำให้ผู้เรียนจากภาษาอื่น (เช่น Python ที่มี `"a,b,c".split(",")` ในตัว) ประหลาดใจ
คือ **มาตรฐาน C++ ไม่มีฟังก์ชัน split string ในตัวจนถึง C++17** และแม้แต่ C++20 ก็ให้แค่
`std::views::split` ซึ่งทำงานผ่าน Ranges (Part 74) และใช้งานอ้อมกว่าที่คิดพอสมควร
(คืนค่าเป็น range ของ subrange ไม่ใช่ `vector<string>` ตรงๆ) ในโลกจริงนักพัฒนา C++
มักเขียนฟังก์ชัน split/join เอง หรือพึ่งพา library ภายนอกอย่าง Boost

```cpp
#include <iostream>
#include <sstream>
#include <string>
#include <vector>

// split ด้วยมือ วิธีที่ 1: ใช้ find() ไล่หาตัวคั่นทีละตัว แล้วตัดด้วย substr()
std::vector<std::string> split(const std::string& text, char delimiter) {
    std::vector<std::string> tokens;
    std::size_t start = 0;
    std::size_t pos;

    while ((pos = text.find(delimiter, start)) != std::string::npos) {
        tokens.push_back(text.substr(start, pos - start));
        start = pos + 1;
    }
    tokens.push_back(text.substr(start));  // ส่วนสุดท้ายหลังตัวคั่นตัวสุดท้าย

    return tokens;
}

// split ด้วยมือ วิธีที่ 2: ใช้ std::stringstream กับ std::getline (นิยมพอๆ กัน อ่านง่ายกว่า)
std::vector<std::string> split_with_stream(const std::string& text, char delimiter) {
    std::vector<std::string> tokens;
    std::stringstream stream(text);
    std::string token;

    while (std::getline(stream, token, delimiter)) {
        tokens.push_back(token);
    }
    return tokens;
}

// join ด้วยมือ: รวม vector<string> กลับเป็น string เดียวโดยมีตัวคั่นระหว่างแต่ละส่วน
std::string join(const std::vector<std::string>& parts, const std::string& delimiter) {
    std::string result;
    for (std::size_t i = 0; i < parts.size(); ++i) {
        result += parts[i];
        if (i + 1 < parts.size()) {
            result += delimiter;
        }
    }
    return result;
}

int main() {
    std::string csv_line = "Somchai,25,Bangkok,Engineer";

    std::vector<std::string> fields = split(csv_line, ',');
    std::cout << "split() ด้วย find/substr:\n";
    for (const auto& field : fields) {
        std::cout << "  [" << field << "]\n";
    }

    std::vector<std::string> fields2 = split_with_stream(csv_line, ',');
    std::cout << "\nsplit_with_stream() ด้วย stringstream:\n";
    for (const auto& field : fields2) {
        std::cout << "  [" << field << "]\n";
    }

    std::string rejoined = join(fields, " | ");
    std::cout << "\njoin() กลับด้วยตัวคั่น \" | \": " << rejoined << "\n";

    return 0;
}
```

ผลลัพธ์:

```
split() ด้วย find/substr:
  [Somchai]
  [25]
  [Bangkok]
  [Engineer]

split_with_stream() ด้วย stringstream:
  [Somchai]
  [25]
  [Bangkok]
  [Engineer]

join() กลับด้วยตัวคั่น " | ": Somchai | 25 | Bangkok | Engineer
```

### เปรียบเทียบสองวิธีของ split

| วิธี | ข้อดี | ข้อเสีย |
|---|---|---|
| `find`/`substr` (manual) | ควบคุมได้ละเอียด, ไม่มี overhead ของ stream | โค้ดยาวกว่า, ต้องระวังการจัดการ index เอง |
| `stringstream`/`getline` | โค้ดสั้น อ่านง่าย | มี overhead ของการสร้าง stream object, ช้ากว่าเล็กน้อยสำหรับข้อมูลจำนวนมาก |

> **เกร็ดความรู้**: ถ้าโปรเจกต์อนุญาตให้ใช้ library ภายนอก `boost::split` (จาก
> `boost/algorithm/string.hpp`) เป็นตัวเลือกที่ครบเครื่องและเร็วกว่าโค้ด manual ทั่วไป
> แต่การเข้าใจวิธีเขียน split/join เองด้วยมือแบบนี้สำคัญมาก เพราะสะท้อนความเข้าใจพื้นฐาน
> เรื่อง `find`/`substr` ซึ่งเป็นทักษะที่ใช้ได้ในทุกสถานการณ์ ไม่ต้องพึ่งพา library ใดๆ เลย

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ต่อ `const char*` สองตัวด้วย `operator+` โดยตรง** — `"Hello, " + "Somchai"` compile
   ไม่ผ่าน เพราะ `operator+` ที่ต่อ string ได้เป็นของ `std::string` ต้องมี `std::string`
   อย่างน้อยหนึ่งฝั่งของนิพจน์เสมอ

2. **คืนค่า `string_view` จากฟังก์ชันที่ตัวแปรต้นทางเป็น local variable** — เป็นสาเหตุอันดับ
   หนึ่งของ dangling `string_view` (ดู 65.5) วิธีป้องกันที่ง่ายที่สุดคือถามตัวเองเสมอว่า
   "ข้อมูลที่ view นี้ชี้ไปจะยังมีชีวิตอยู่หลังฟังก์ชันนี้ return หรือไม่"

3. **เก็บ `string_view` ไว้ข้าม scope ที่ `std::string` ต้นทางอาจถูกแก้ไข (mutate)** —
   การ `+=`, `insert`, `resize` ใดๆ ที่ทำให้ `std::string` ต้องขยาย buffer (reallocate)
   จะทำให้ `string_view` เดิมที่ชี้ไปยัง buffer เก่ากลายเป็น dangling ทันที แม้ว่า
   `std::string` ต้นทางจะยังไม่ถูกทำลายก็ตาม

4. **ใช้ `operator[]` แทน `.at()` เมื่อไม่แน่ใจว่า index ปลอดภัยหรือไม่** — `operator[]`
   ไม่ตรวจสอบขอบเขต การเข้าถึง index ที่เกินขนาดเป็น Undefined Behavior ในขณะที่ `.at()`
   จะ throw `std::out_of_range` ให้ดักจับได้ ควรใช้ `.at()` เมื่อ index มาจาก input
   ภายนอกที่ยังไม่ได้ validate

5. **สร้าง `std::regex` ใหม่ทุกครั้งในลูป** — pattern compilation มีค่าใช้จ่ายสูง ควรสร้าง
   `std::regex` ไว้ครั้งเดียว (เช่นเป็น `static const` ในฟังก์ชัน หรือสร้างไว้นอกลูป)
   แล้วนำมาใช้ซ้ำ

6. **เข้าใจผิดว่า `regex_match` เท่ากับ `regex_search`** — `regex_match` ต้องการให้ **ทั้ง
   string** ตรงกับ pattern พอดี ในขณะที่ `regex_search` แค่หาว่ามี **บางส่วน** ตรงหรือไม่
   การใช้ผิดตัวเป็นสาเหตุของบั๊กเรื่อง validation ที่พบบ่อยมาก (เช่น validate email
   ด้วย `regex_search` โดยไม่ใส่ `^...$` อาจปล่อยผ่านอีเมลที่มีขยะปนอยู่หน้า/หลังได้)

---

## แบบฝึกหัดท้ายบท

1. เขียนฟังก์ชัน `std::string my_reverse(const std::string& input)` ที่กลับข้อความ
   (reverse) เอง โดยห้ามใช้ `std::reverse` หรือฟังก์ชัน built-in ใดๆ ที่ทำหน้าที่นี้ตรงๆ
2. เขียนฟังก์ชัน `bool is_palindrome(std::string_view text)` ที่ตรวจสอบว่าข้อความที่ให้มา
   เป็น palindrome (อ่านจากหน้าไปหลังกับหลังไปหน้าเหมือนกัน) หรือไม่ โดยใช้ `string_view`
   เป็นพารามิเตอร์
3. เขียนฟังก์ชัน `bool is_valid_thai_phone(const std::string& phone)` ที่ตรวจสอบว่า string
   ที่ให้มาตรงกับรูปแบบเบอร์มือถือไทย (`0XX-XXX-XXXX` หรือ `0XXXXXXXXX` ติดกัน 10 หลัก)
   โดยใช้ `std::regex`
4. เขียนฟังก์ชัน `std::string_view trim(std::string_view text)` ที่ตัดช่องว่าง (space,
   tab, newline) ที่อยู่หน้าและหลังข้อความออก โดยใช้ `find_first_not_of` /
   `find_last_not_of`
5. เขียนโปรแกรมทดลองสร้าง `std::string` ที่มีความยาวเพิ่มขึ้นทีละ 1 ตัวอักษร (ตั้งแต่ 1 ถึง
   30 ตัวอักษร) แล้วใช้เทคนิคจาก 65.3 ตรวจสอบว่าที่ความยาวเท่าไหร่บนเครื่องของผู้เรียนเอง
   ที่ `std::string` เปลี่ยนจาก SSO ไปเป็น heap allocation
6. เขียนฟังก์ชัน `std::string join_views(const std::vector<std::string_view>& parts, std::string_view delimiter)`
   ที่ทำหน้าที่เหมือน `join` ใน 65.8 แต่รับ `vector<string_view>` แทน `vector<string>`
   แล้วอธิบายว่าทำไมฟังก์ชันนี้ต้อง**คืนค่าเป็น `std::string`** ไม่ใช่ `std::string_view`

### แนวทางเฉลยข้อ 1 และ 2

```cpp
#include <iostream>
#include <string>
#include <string_view>

// ข้อ 1: reverse string เองโดยไม่เรียก std::reverse หรือ built-in ใดๆ
std::string my_reverse(const std::string& input) {
    std::string result(input.size(), '\0');  // สร้าง string ขนาดเท่ากัน เติมด้วย '\0' ไปก่อน
    for (std::size_t i = 0; i < input.size(); ++i) {
        result[i] = input[input.size() - 1 - i];
    }
    return result;
}

// ข้อ 2: ตรวจสอบ palindrome (คำที่อ่านจากหน้าไปหลังหรือหลังไปหน้าแล้วเหมือนกัน)
// ใช้ string_view เพื่อไม่ต้อง copy ข้อมูล
bool is_palindrome(std::string_view text) {
    std::size_t left = 0;
    std::size_t right = text.size() > 0 ? text.size() - 1 : 0;
    while (left < right) {
        if (text[left] != text[right]) {
            return false;
        }
        ++left;
        --right;
    }
    return true;
}

int main() {
    std::string s = "Hello, World!";
    std::cout << "original: " << s << "\n";
    std::cout << "reversed: " << my_reverse(s) << "\n\n";

    std::string_view words[] = {"level", "racecar", "hello", "a", ""};
    for (auto w : words) {
        std::cout << "\"" << w << "\" เป็น palindrome หรือไม่: "
                  << std::boolalpha << is_palindrome(w) << "\n";
    }

    return 0;
}
```

ผลลัพธ์:

```
original: Hello, World!
reversed: !dlroW ,olleH

"level" เป็น palindrome หรือไม่: true
"racecar" เป็น palindrome หรือไม่: true
"hello" เป็น palindrome หรือไม่: false
"a" เป็น palindrome หรือไม่: true
"" เป็น palindrome หรือไม่: true
```

**คำอธิบาย**: `my_reverse` สร้าง `std::string result` ขนาดเท่า input ล่วงหน้าด้วย
constructor `std::string(count, ch)` แล้ว copy ตัวอักษรจากท้ายไปหน้าทีละตัว
`is_palindrome` ใช้ two-pointer technique (`left` วิ่งจากหน้า, `right` วิ่งจากหลัง)
เทียบตัวอักษรเข้าหากันจนกว่าจะพบตัวที่ไม่ตรงกัน (คืน `false` ทันที) หรือ pointer ทั้งสอง
มาบรรจบกัน (คืน `true`) — สังเกตว่า `is_palindrome` รับ `string_view` เพราะฟังก์ชันนี้
แค่ "อ่าน" ข้อมูล ไม่ต้องเก็บไว้ใช้ต่อ จึงไม่จำเป็นต้อง copy เป็น `std::string` เลย

### แนวทางเฉลยข้อ 3 และ 4

```cpp
#include <iostream>
#include <regex>
#include <string>
#include <string_view>
#include <vector>

// ข้อ 3: ตรวจสอบเบอร์โทรศัพท์มือถือไทย รูปแบบ 0XX-XXX-XXXX หรือ 0XXXXXXXXX (10 หลักติดกัน)
bool is_valid_thai_phone(const std::string& phone) {
    static const std::regex pattern(R"(^0\d{2}-?\d{3}-?\d{4}$)");
    return std::regex_match(phone, pattern);
}

// ข้อ 4: ตัดช่องว่าง (whitespace) หน้า-หลังออกจาก string ด้วย string_view (ไม่ copy จนกว่าจำเป็น)
std::string_view trim(std::string_view text) {
    const char* whitespace = " \t\n\r";
    std::size_t start = text.find_first_not_of(whitespace);
    if (start == std::string_view::npos) {
        return "";  // string ว่างล้วนแต่ whitespace
    }
    std::size_t end = text.find_last_not_of(whitespace);
    return text.substr(start, end - start + 1);
}

int main() {
    std::vector<std::string> phones = {
        "081-234-5678", "0812345678", "02-123-4567", "12-345-6789", "081-234-56789",
    };
    for (const auto& p : phones) {
        std::cout << p << " -> " << (is_valid_thai_phone(p) ? "ถูกต้อง" : "ไม่ถูกต้อง") << "\n";
    }

    std::cout << "\n";
    std::string_view padded = "   Hello, World!   ";
    std::string_view trimmed = trim(padded);
    std::cout << "ก่อน trim: [" << padded << "]\n";
    std::cout << "หลัง trim: [" << trimmed << "]\n";

    return 0;
}
```

ผลลัพธ์:

```
081-234-5678 -> ถูกต้อง
0812345678 -> ถูกต้อง
02-123-4567 -> ไม่ถูกต้อง
12-345-6789 -> ไม่ถูกต้อง
081-234-56789 -> ไม่ถูกต้อง

ก่อน trim: [   Hello, World!   ]
หลัง trim: [Hello, World!]
```

**คำอธิบาย**: pattern `^0\d{2}-?\d{3}-?\d{4}$` หมายถึง "เริ่มต้นด้วย `0` ตามด้วยตัวเลข 2
ตัว ตามด้วยขีดกลาง (มีหรือไม่มีก็ได้ จาก `-?`) ตามด้วยตัวเลข 3 ตัว ตามด้วยขีดกลางอีกครั้ง
(มีหรือไม่มีก็ได้) แล้วจบด้วยตัวเลข 4 ตัวพอดี" — สังเกตว่า `"02-123-4567"` (เบอร์บ้าน 9
หลัก) ถูกตัดสินว่า "ไม่ถูกต้อง" เพราะ pattern นี้ตรวจเฉพาะรูปแบบเบอร์มือถือ 10 หลักเท่านั้น
ซึ่งเป็นพฤติกรรมที่ถูกต้องตามที่โจทย์กำหนด ส่วน `trim` ใช้ `find_first_not_of` หา index
ของตัวอักษรตัวแรกที่ไม่ใช่ whitespace และ `find_last_not_of` หาตัวสุดท้าย แล้ว `substr`
ตัดเฉพาะช่วงกลางออกมาเป็น `string_view` ใหม่ — ทั้งหมดนี้ไม่มีการ copy ข้อมูลต้นทางเลย
แม้แต่ไบต์เดียว เพียงแค่ขยับ pointer และปรับความยาวของ view เท่านั้น

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เจาะลึก `std::string` เปรียบเทียบกับ C-string จาก Part 7 เห็นชัดว่าการจัดการหน่วยความจำ
  อัตโนมัติช่วยลดบั๊กประเภท buffer overflow ได้มากแค่ไหน
- ฝึกใช้เมธอดหลักของ `std::string`: `operator+`, `substr`, `find`, `replace`
- เข้าใจ Small String Optimization (SSO) และเห็นด้วยโค้ดจริงว่า string สั้นๆ ไม่ต้องขอ
  heap memory เลย
- รู้จัก `std::string_view` (C++17) ว่าเป็น non-owning view ที่เร็วกว่าเพราะไม่ copy ข้อมูล
  พร้อมเข้าใจอันตรายสำคัญที่สุดของมันคือ dangling view และวิธีป้องกัน
- ใช้ `std::regex` เบื้องต้นได้จริง: `regex_match`, `regex_search`, `regex_replace`
  พร้อมตัวอย่างการ validate อีเมลและเบอร์โทรศัพท์
- เขียนฟังก์ชัน split/join ด้วยมือ และเข้าใจว่าทำไม C++ มาตรฐานถึงไม่มีให้ใช้ตรงๆ

ใน **Part 66** เราจะเจาะลึกกลไกเบื้องหลังที่ container ของ STL ใช้จัดสรรและคืนหน่วยความจำ
นั่นคือ **Custom Allocator** — จะเห็นว่า `std::allocator` เริ่มต้นทำงานอย่างไร และทำไม
บางครั้งวิศวกรถึงต้องเขียน allocator ของตัวเองสำหรับงานที่ต้องการประสิทธิภาพสูงสุด

**ต่อไป:** [Part 66 — Custom Allocator ใน STL](./part-066-custom-allocators.md)
