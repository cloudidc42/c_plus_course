# Part 107: การจัดการ JSON ด้วย nlohmann/json (Step 849–856)

> Module I — Web Development ด้วย C/C++ | Part 107 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 849–856
> Part ก่อนหน้า: [Part 106 — เชื่อมต่อ PostgreSQL/MySQL ด้วย C++](./part-106-postgresql-mysql-cpp.md) | Part ถัดไป: [Part 108 — โปรเจกต์: REST API CRUD ครบวงจร](./part-108-rest-api-crud-project.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม **JSON (JavaScript Object Notation)** ถึงกลายเป็นมาตรฐานการแลกเปลี่ยน
   ข้อมูลของ Web API เกือบทุกระบบในปัจจุบัน
2. ติดตั้งและใช้งานไลบรารี **nlohmann/json** ซึ่งเป็น header-only library ที่ทำให้ C++ จัดการ
   JSON ได้ง่ายในระดับใกล้เคียงกับภาษาสคริปต์อย่าง Python หรือ JavaScript
3. สร้าง JSON object และ array ด้วยมือผ่าน syntax ที่กระชับของไลบรารีนี้
4. Parse JSON string จากภายนอกเข้ามาเป็น object แล้วเข้าถึงค่าด้วย `operator[]`, `.at()`,
   และ `.value()` พร้อมเข้าใจข้อแตกต่างเชิงความปลอดภัยของแต่ละวิธี
5. แปลง C++ struct เป็น/จาก JSON โดยอัตโนมัติด้วย `NLOHMANN_DEFINE_TYPE_INTRUSIVE` โดยไม่ต้อง
   เขียนโค้ดแปลงเองแม้แต่บรรทัดเดียว
6. จัดการ error ที่เกิดจาก JSON ผิดรูปแบบได้อย่างถูกต้อง ผ่าน exception hierarchy ของ
   nlohmann/json (`parse_error`, `out_of_range`, `type_error`)
7. เชื่อมโยงความรู้จาก Part 105/106 เข้ากับ Part นี้ โดยแปลงผลลัพธ์จาก database query ให้
   กลายเป็น JSON response สำหรับ REST API ที่พร้อมส่งกลับให้ client ได้จริง

---

## 107.1 ทำไม JSON ถึงสำคัญมากสำหรับ Web API (Step 849)

ใน Part 102–104 เราสร้าง REST API endpoint ด้วย Crow และ Pistache แล้ว และใน Part 105–106
เราเชื่อมต่อฐานข้อมูลเพื่อเก็บข้อมูลถาวรได้แล้ว แต่ยังขาดจิ๊กซอว์ชิ้นสำคัญชิ้นหนึ่ง: **เมื่อ
Web Server ต้องส่งข้อมูลกลับไปให้ client (เบราว์เซอร์, mobile app, หรือระบบอื่น) ควรส่งข้อมูล
ในรูปแบบไหน?**

คำตอบที่โลกซอฟต์แวร์ยอมรับร่วมกันมานานคือ **JSON** ด้วยเหตุผลหลักๆ ดังนี้:

- **Human-readable**: เป็น text ธรรมดาที่มนุษย์อ่านและเข้าใจโครงสร้างได้ทันที ต่างจากรูปแบบ
  binary อย่าง Protocol Buffers ที่ต้องมี schema มาช่วยตีความ
- **ภาษาอิสระ (Language-agnostic)**: แทบทุกภาษาโปรแกรมมี library แปลง JSON <-> โครงสร้างข้อมูล
  ของตัวเองอยู่แล้ว (Python: `dict`, JavaScript: object literal โดยตรง, Java: `Map`/POJO,
  C++: อย่างที่จะเรียนใน Part นี้) ทำให้ frontend ที่เขียนด้วย JavaScript และ backend ที่เขียน
  ด้วย C++ คุยกันได้โดยไม่ต้องสนใจว่าอีกฝั่งใช้ภาษาอะไร
- **โครงสร้างเรียบง่ายแต่ยืดหยุ่น**: รองรับ object (`{}`), array (`[]`), string, number,
  boolean, `null` และซ้อนกันได้ไม่จำกัดชั้น เพียงพอสำหรับข้อมูลเกือบทุกรูปแบบที่ REST API
  ต้องการส่ง
- **เป็นมาตรฐานของ HTTP body**: `Content-Type: application/json` เป็น header ที่ REST API
  เกือบทุกตัวในโลกใช้ร่วมกัน

ตัวอย่าง JSON response ทั่วไปที่ REST API ส่งกลับ:

```json
{
  "status": "ok",
  "data": {
    "id": 1,
    "name": "Somchai",
    "email": "somchai@example.com"
  }
}
```

ปัญหาคือ **C++ ไม่มี JSON เป็นส่วนหนึ่งของภาษาหรือ Standard Library มาตั้งแต่ต้น** (ต่างจาก
JavaScript ที่ JSON คือ subset ของ syntax ภาษาเอง) เราจึงต้องใช้ไลบรารีภายนอก และไลบรารีที่
ได้รับความนิยมสูงสุดในโลก C++ คือ **nlohmann/json**

---

## 107.2 ติดตั้งและทำความรู้จัก nlohmann/json (Step 850)

nlohmann/json เขียนโดย Niels Lohmann เป็น **header-only library** หมายความว่าไม่ต้อง compile
หรือ link เป็นไฟล์ `.so`/`.a` แยก แค่ `#include` ไฟล์ header เดียวก็ใช้งานได้ทันที

### ติดตั้งบน Ubuntu/Debian

```bash
sudo apt install nlohmann-json3-dev -y
```

ตรวจสอบว่าไฟล์ header ถูกติดตั้งไว้ที่ไหน:

```bash
find /usr/include -iname "json.hpp"
```

ผลลัพธ์บนเครื่องที่ใช้เขียนบทเรียนนี้:

```
/usr/include/nlohmann/json.hpp
```

### ทำไม syntax ถึงคล้าย Python dict

nlohmann/json ออกแบบมาให้ใช้งานสะดวกที่สุดเท่าที่ C++ จะทำได้ โดยใช้ประโยชน์จาก
`std::initializer_list`, `operator[]` overloading, และ template magic ต่างๆ เพื่อให้ syntax
ออกมากระชับใกล้เคียงกับภาษาที่มี native support สำหรับ JSON เช่น Python:

```python
# Python
person = {"name": "Somchai", "age": 25}
print(person["name"])
```

```cpp
// C++ ด้วย nlohmann/json
json person = {{"name", "Somchai"}, {"age", 25}};
std::cout << person["name"];
```

ทั้งหมดนี้ทำได้โดยที่ `json` เป็นเพียง **type เดียว** (`nlohmann::json`) ที่เก็บได้ทั้ง object,
array, string, number, boolean หรือ `null` — สลับชนิดกันได้แบบ dynamic typing เหมือนตัวแปร
ในภาษาสคริปต์ ทั้งที่ C++ เป็นภาษา statically-typed โดยพื้นฐาน (เบื้องหลังใช้เทคนิค type
erasure ที่คล้ายกับ `std::variant`/`std::any` ที่เคยเรียนใน Module D)

ตลอด Part นี้เราจะใช้ alias มาตรฐานที่เอกสารของไลบรารีแนะนำ:

```cpp
#include <nlohmann/json.hpp>
using json = nlohmann::json;
```

---

## 107.3 สร้าง JSON Object/Array ด้วยมือ (Step 851)

```cpp
// 01_json_basics.cpp - สร้าง JSON object/array ด้วยมือ และแปลงเป็น string
#include <nlohmann/json.hpp>
#include <iostream>

using json = nlohmann::json;

int main(void) {
    // สร้าง object แบบ initializer list คล้าย Python dict literal
    json person = {
        {"name", "Somchai"},
        {"age", 25},
        {"is_student", false},
        {"gpa", 3.75}
    };

    // เพิ่ม field ทีหลังได้เหมือน map ปกติ
    person["email"] = "somchai@example.com";

    // สร้าง array
    person["skills"] = {"C++", "Python", "SQL"};

    // สร้าง nested object
    person["address"] = {
        {"city", "Bangkok"},
        {"zipcode", "10110"}
    };

    std::cout << "-- dump() แบบบรรทัดเดียว --\n";
    std::cout << person.dump() << "\n\n";

    std::cout << "-- dump(4) แบบจัด indent สวยงาม --\n";
    std::cout << person.dump(4) << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 01_json_basics.cpp -o 01_json_basics
./01_json_basics
```

ผลลัพธ์จริง:

```
-- dump() แบบบรรทัดเดียว --
{"address":{"city":"Bangkok","zipcode":"10110"},"age":25,"email":"somchai@example.com","gpa":3.75,"is_student":false,"name":"Somchai","skills":["C++","Python","SQL"]}

-- dump(4) แบบจัด indent สวยงาม --
{
    "address": {
        "city": "Bangkok",
        "zipcode": "10110"
    },
    "age": 25,
    "email": "somchai@example.com",
    "gpa": 3.75,
    "is_student": false,
    "name": "Somchai",
    "skills": [
        "C++",
        "Python",
        "SQL"
    ]
}
```

### สังเกตสิ่งสำคัญ

- `person["email"] = "somchai@example.com";` — ใช้ `operator[]` เพิ่ม field ใหม่ได้ทันทีแม้
  ยังไม่มี key นี้อยู่ก่อน (พฤติกรรมนี้เหมือน `std::map::operator[]`)
- `person["skills"] = {"C++", "Python", "SQL"};` — nlohmann/json ตีความ `initializer_list`
  ของ string ล้วนๆ (ไม่มี key-value pair) ว่าเป็น **JSON array** โดยอัตโนมัติ
- `dump()` แปลง `json` object กลับเป็น `std::string` — เรียกโดยไม่ใส่ argument จะได้ string
  บรรทัดเดียวไม่มีช่องว่าง (เหมาะกับส่งผ่าน network เพื่อประหยัด bandwidth) ส่วน `dump(4)`
  จะจัด indent 4 ช่องว่างให้อ่านง่าย (เหมาะกับ debug หรือ log)
- **ลำดับ key ในผลลัพธ์ `dump()` เรียงตามตัวอักษร** ไม่ใช่ลำดับที่เราใส่ตอนสร้าง เพราะ
  `nlohmann::json` เก็บ object ด้วย `std::map` ภายใน (เรียงตาม key) เป็นค่า default

---

## 107.4 Parse JSON String และเข้าถึงค่า (Step 852)

สถานการณ์ที่พบบ่อยที่สุดในการเขียน Web API คือการรับ JSON ที่ client ส่งมา (เช่น request
body ของ `POST`/`PUT`) เป็น `std::string` แล้วต้อง **parse** ให้กลายเป็น `json` object ที่
เข้าถึงค่าข้างในได้

```cpp
// 02_json_parse.cpp - parse JSON string เข้ามาเป็น object และเข้าถึงค่า
#include <nlohmann/json.hpp>
#include <iostream>
#include <string>

using json = nlohmann::json;

int main(void) {
    std::string raw = R"({
        "id": 42,
        "name": "Suda",
        "score": 92.5,
        "tags": ["honor", "top10"],
        "active": true
    })";

    json j = json::parse(raw);

    // อ่านค่าด้วย operator[] (ถ้า key ไม่มีจะคืนค่า null แทนการ throw)
    std::cout << "id (operator[]) = " << j["id"] << '\n';

    // อ่านค่าด้วย .at() (ถ้า key ไม่มีจะ throw exception -- ปลอดภัยกว่าเมื่อ parse ข้อมูลจากภายนอก)
    std::cout << "name (.at())    = " << j.at("name").get<std::string>() << '\n';

    // แปลงชนิดข้อมูลชัดเจนด้วย .get<T>()
    double score = j.at("score").get<double>();
    bool active = j.at("active").get<bool>();
    std::cout << "score           = " << score << '\n';
    std::cout << "active          = " << std::boolalpha << active << '\n';

    // วนลูปใน array
    std::cout << "tags:\n";
    for (const auto& tag : j.at("tags")) {
        std::cout << "  - " << tag.get<std::string>() << '\n';
    }

    // ตรวจสอบว่ามี key อยู่หรือไม่ก่อนอ่าน (ปลอดภัยกว่าการเดา)
    if (j.contains("nickname")) {
        std::cout << "nickname = " << j["nickname"] << '\n';
    } else {
        std::cout << "ไม่มี key \"nickname\" ใน JSON นี้\n";
    }

    // ใช้ value() เพื่อกำหนดค่า default เมื่อ key ไม่มี (สะดวกกว่า contains() + operator[])
    std::string nickname = j.value("nickname", "(ไม่ระบุ)");
    std::cout << "nickname (with default) = " << nickname << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 02_json_parse.cpp -o 02_json_parse
./02_json_parse
```

ผลลัพธ์จริง:

```
id (operator[]) = 42
name (.at())    = Suda
score           = 92.5
active          = true
tags:
  - honor
  - top10
ไม่มี key "nickname" ใน JSON นี้
nickname (with default) = (ไม่ระบุ)
```

### เปรียบเทียบวิธีเข้าถึงค่าทั้ง 3 แบบ

| วิธี | พฤติกรรมเมื่อ key ไม่มีอยู่ | พฤติกรรมเมื่อ key มีแต่ชนิดไม่ตรง | ควรใช้เมื่อ |
|---|---|---|---|
| `j["key"]` (`operator[]`) | **สร้าง key ใหม่เป็น `null` ให้อัตโนมัติ** (ไม่ throw) | ต้อง `.get<T>()` แยกอีกที ถึงจะ throw ถ้าไม่ตรง | รู้แน่ชัดแล้วว่า key มีอยู่จริง หรือกำลัง "สร้าง" JSON ใหม่ (write) |
| `j.at("key")` | **throw `json::out_of_range`** ทันที | throw `json::type_error` เมื่อ `.get<T>()` | อ่านค่า (read) จาก JSON ที่มาจากภายนอกซึ่งไม่แน่ใจโครงสร้าง 100% |
| `j.value("key", default_value)` | คืนค่า `default_value` แบบเงียบๆ ไม่ throw | throw `json::type_error` ถ้า key มีอยู่แต่แปลงชนิดไม่ได้ | field ที่เป็น optional และมีค่า default ที่สมเหตุสมผล |

> ข้อควรระวังสำคัญ: การใช้ `j["key"]` กับ `json` ที่เป็น `const` จะไม่สร้าง key ใหม่ (เพราะ
> แก้ไขไม่ได้) แต่จะ throw แทนถ้า key ไม่มี ทำให้พฤติกรรมของ `operator[]` **ต่างกันระหว่าง
> const กับ non-const object** นี่คือเหตุผลสำคัญที่หลายทีมกำหนดเป็น convention ว่า **ให้ใช้
> `.at()` เสมอเมื่ออ่านค่าจาก JSON ที่ parse มาจากภายนอก** เพื่อไม่ให้พฤติกรรมสับสน

---

## 107.5 Error Handling เมื่อ JSON ผิดรูปแบบ (Step 853)

เมื่อ JSON มาจากภายนอกโปรแกรม (เช่น request body ของ HTTP, ไฟล์ config, หรือ response จาก
API อื่น) มันสามารถผิดรูปแบบได้เสมอ — nlohmann/json มี exception hierarchy ที่ชัดเจนสำหรับ
จัดการสถานการณ์เหล่านี้:

```cpp
// 03_json_errors.cpp - error handling เมื่อ JSON ผิด format หรือ key ไม่มี/ชนิดไม่ตรง
#include <nlohmann/json.hpp>
#include <iostream>

using json = nlohmann::json;

int main(void) {
    // 1) json::parse_error - เมื่อ string ไม่ใช่ JSON ที่ถูกต้องตามไวยากรณ์
    std::string broken = R"({"name": "Somchai", "age": })"; // ค่าของ "age" หายไป
    try {
        json j = json::parse(broken);
        std::cout << "parse สำเร็จ (ไม่ควรมาถึงบรรทัดนี้)\n";
    } catch (const json::parse_error& e) {
        std::cout << "[parse_error] " << e.what() << '\n';
        std::cout << "  byte offset ที่ผิด: " << e.byte << "\n\n";
    }

    // 2) json::out_of_range - เมื่อเรียก .at() กับ key ที่ไม่มีอยู่จริง
    json j2 = {{"name", "Suda"}};
    try {
        std::string email = j2.at("email").get<std::string>(); // key "email" ไม่มี
        std::cout << email << '\n';
    } catch (const json::out_of_range& e) {
        std::cout << "[out_of_range] " << e.what() << "\n\n";
    }

    // 3) json::type_error - เมื่อพยายาม .get<T>() ผิดชนิด
    json j3 = {{"age", "twenty-five"}}; // เก็บเป็น string ทั้งที่ควรเป็นตัวเลข
    try {
        int age = j3.at("age").get<int>();
        std::cout << age << '\n';
    } catch (const json::type_error& e) {
        std::cout << "[type_error] " << e.what() << "\n\n";
    }

    // แนวทางที่ปลอดภัยเมื่อรับ JSON จากภายนอก (เช่น request body ของ REST API):
    // ห่อการ parse+extract ทั้งหมดด้วย try/catch เดียว แล้วคืน error response ที่เหมาะสม
    auto safe_extract_name = [](const std::string& body) -> std::string {
        try {
            json j = json::parse(body);
            return j.at("name").get<std::string>();
        } catch (const json::exception& e) {
            // json::exception คือ base class ของ parse_error, out_of_range, type_error ทั้งหมด
            return std::string("(ข้อมูลไม่ถูกต้อง: ") + e.what() + ")";
        }
    };

    std::cout << "safe_extract_name(broken)      = " << safe_extract_name(broken) << '\n';
    std::cout << "safe_extract_name(valid)       = "
               << safe_extract_name(R"({"name":"Anan"})") << '\n';
    std::cout << "safe_extract_name(missing key) = "
               << safe_extract_name(R"({"nickname":"Anan"})") << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 03_json_errors.cpp -o 03_json_errors
./03_json_errors
```

ผลลัพธ์จริง:

```
[parse_error] [json.exception.parse_error.101] parse error at line 1, column 28: syntax error while parsing value - unexpected '}'; expected '[', '{', or a literal
  byte offset ที่ผิด: 28

[out_of_range] [json.exception.out_of_range.403] key 'email' not found

[type_error] [json.exception.type_error.302] type must be number, but is string

safe_extract_name(broken)      = (ข้อมูลไม่ถูกต้อง: [json.exception.parse_error.101] parse error at line 1, column 28: syntax error while parsing value - unexpected '}'; expected '[', '{', or a literal)
safe_extract_name(valid)       = Anan
safe_extract_name(missing key) = (ข้อมูลไม่ถูกต้อง: [json.exception.out_of_range.403] key 'name' not found)
```

### Exception Hierarchy ของ nlohmann/json

```
json::exception                 (base class ของทั้งหมด)
   ├── json::parse_error        (JSON syntax ผิด — เกิดตอน json::parse())
   ├── json::type_error         (พยายาม .get<T>() หรือดำเนินการผิดชนิด)
   ├── json::out_of_range       (เรียก .at() กับ index/key ที่ไม่มี)
   ├── json::invalid_iterator   (ใช้ iterator ผิดวิธี)
   └── json::other_error        (กรณีอื่นๆ)
```

การจับ `json::exception` (base class) เพียงตัวเดียวใน `catch` จึงครอบคลุมทุก error ที่มาจาก
การประมวลผล JSON ได้ในจุดเดียว — เป็นรูปแบบที่นิยมมากในโค้ดฝั่ง server เมื่อรับ request body
จาก client เพราะไม่ว่า client จะส่ง JSON ผิดแบบไหน (syntax ผิด, ขาด field, ชนิดข้อมูลไม่ตรง)
เราต้องการทำสิ่งเดียวกันเสมอคือ **ตอบกลับ HTTP 400 Bad Request** พร้อมข้อความอธิบาย

---

## 107.6 แปลง Struct เป็น/จาก JSON อัตโนมัติ (Step 854)

การเขียนโค้ดแปลงค่าจาก `json` object ไปเป็น struct ทีละ field แบบในหัวข้อก่อนหน้าจะน่าเบื่อ
มากเมื่อ struct มีหลาย field และมีหลาย struct ในโปรเจกต์ nlohmann/json มี macro
**`NLOHMANN_DEFINE_TYPE_INTRUSIVE`** ที่สร้างฟังก์ชัน `to_json()`/`from_json()` ให้อัตโนมัติ

```cpp
// 04_json_struct.cpp - แปลง C++ struct เป็น/จาก JSON อัตโนมัติด้วย NLOHMANN_DEFINE_TYPE_INTRUSIVE
#include <nlohmann/json.hpp>
#include <iostream>
#include <vector>

using json = nlohmann::json;

struct Employee {
    int id;
    std::string name;
    std::string department;
    double salary;

    // Macro นี้จะสร้างฟังก์ชัน to_json() และ from_json() ให้อัตโนมัติ โดยแมป field
    // ทุกตัวที่ระบุเข้ากับ key ชื่อเดียวกันใน JSON - ไม่ต้องเขียนโค้ดแปลงเองเลยสักบรรทัด
    NLOHMANN_DEFINE_TYPE_INTRUSIVE(Employee, id, name, department, salary)
};

int main(void) {
    Employee e1{1, "Somchai", "Engineering", 48000.0};

    // struct -> json ทำได้ตรงๆ ด้วยการ assign
    json j = e1;
    std::cout << "Employee -> JSON:\n" << j.dump(2) << "\n\n";

    // json -> struct ทำได้ตรงๆ ด้วย .get<T>()
    std::string raw = R"({"id":2,"name":"Suda","department":"Marketing","salary":38000.0})";
    json j2 = json::parse(raw);
    Employee e2 = j2.get<Employee>();
    std::cout << "JSON -> Employee: id=" << e2.id << " name=" << e2.name
               << " department=" << e2.department << " salary=" << e2.salary << "\n\n";

    // ใช้กับ vector ของ struct ได้ทันที (nlohmann/json รองรับ container มาตรฐานให้อัตโนมัติ)
    std::vector<Employee> employees = {
        {1, "Somchai", "Engineering", 48000.0},
        {2, "Suda", "Marketing", 38000.0},
        {3, "Anan", "Engineering", 52000.0}
    };
    json j_array = employees;
    std::cout << "vector<Employee> -> JSON array:\n" << j_array.dump(2) << "\n\n";

    // แปลงกลับจาก JSON array เป็น vector<Employee>
    std::vector<Employee> parsed_back = j_array.get<std::vector<Employee>>();
    std::cout << "แปลงกลับได้ " << parsed_back.size() << " รายการ, คนแรกชื่อ "
               << parsed_back.front().name << '\n';

    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 04_json_struct.cpp -o 04_json_struct
./04_json_struct
```

ผลลัพธ์จริง:

```
Employee -> JSON:
{
  "department": "Engineering",
  "id": 1,
  "name": "Somchai",
  "salary": 48000.0
}

JSON -> Employee: id=2 name=Suda department=Marketing salary=38000

vector<Employee> -> JSON array:
[
  {
    "department": "Engineering",
    "id": 1,
    "name": "Somchai",
    "salary": 48000.0
  },
  {
    "department": "Marketing",
    "id": 2,
    "name": "Suda",
    "salary": 38000.0
  },
  {
    "department": "Engineering",
    "id": 3,
    "name": "Anan",
    "salary": 52000.0
  }
]

แปลงกลับได้ 3 รายการ, คนแรกชื่อ Somchai
```

### เบื้องหลัง Macro ทำงานอย่างไร

`NLOHMANN_DEFINE_TYPE_INTRUSIVE(Employee, id, name, department, salary)` จะขยายเป็นสมาชิก
ฟังก์ชัน `to_json`/`from_json` ภายใน class `Employee` โดยประมาณเทียบเท่ากับการเขียนเองแบบนี้:

```cpp
// สิ่งที่ macro ทำให้อัตโนมัติ (โดยประมาณ) — ไม่ต้องเขียนเองอีกต่อไป
void to_json(json& j, const Employee& e) {
    j = json{{"id", e.id}, {"name", e.name}, {"department", e.department}, {"salary", e.salary}};
}
void from_json(const json& j, Employee& e) {
    j.at("id").get_to(e.id);
    j.at("name").get_to(e.name);
    j.at("department").get_to(e.department);
    j.at("salary").get_to(e.salary);
}
```

เมื่อไลบรารีเจอฟังก์ชัน `to_json`/`from_json` ที่ match กับ type นั้นๆ (ผ่านกลไก **ADL —
Argument-Dependent Lookup** ที่เคยพูดถึงตอนเรียน operator overloading ใน Module D) มันจะเรียก
ใช้อัตโนมัติทุกครั้งที่มีการ assign `json j = some_struct;` หรือ `some_struct = j.get<T>();`
โดยไม่ต้องเขียนโค้ดเรียกเอง — นี่คือเหตุผลที่ `vector<Employee>` ในตัวอย่างข้างต้นแปลงเป็น
JSON array ได้ทันทีด้วย เพราะ nlohmann/json รู้วิธีแปลง container มาตรฐานอยู่แล้ว (แปลงทีละ
element โดยเรียก `to_json`/`from_json` ของ `Employee` ซ้ำๆ)

> **ข้อจำกัดของ `NLOHMANN_DEFINE_TYPE_INTRUSIVE`**: field ทุกตัวที่ระบุใน macro ถือว่า
> **จำเป็นต้องมี (required)** เสมอตอน `from_json` — ถ้า JSON ที่ parse เข้ามาขาด field ใดไป
> จะ throw `json::out_of_range` ทันที ถ้าต้องการ field ที่ optional ต้องเขียน `from_json`
> เองแบบ manual โดยใช้ `j.value("field", default)` แทน `j.at("field")`

---

## 107.7 ใช้งานร่วมกับ Container มาตรฐานอื่นๆ (Step 855)

ประโยชน์อย่างหนึ่งของการรองรับ `to_json`/`from_json` ผ่าน ADL คือ nlohmann/json รองรับ
container ของ C++ Standard Library แทบทั้งหมดโดยอัตโนมัติ ไม่ว่าจะเป็น `std::vector`,
`std::map`, `std::optional` (ตั้งแต่ C++17), `std::pair` ไปจนถึง type ของเราเองที่ประกาศด้วย
`NLOHMANN_DEFINE_TYPE_INTRUSIVE` เหมือนที่เห็นแล้วในหัวข้อก่อนหน้ากับ `std::vector<Employee>`

รูปแบบที่พบบ่อยมากในการเขียน REST API คือการห่อ array ของ struct ไว้ใน object ที่มี metadata
เพิ่มเติม (เช่น `status`, `count`) ก่อนส่งกลับเป็น response — รูปแบบนี้จะถูกใช้เต็มรูปแบบใน
หัวข้อถัดไปที่เชื่อมกับฐานข้อมูลจริง

```cpp
// ตัวอย่างสั้นๆ: ห่อ vector<Employee> ด้วย status/count ก่อนส่งเป็น response
json response = {
    {"status", "ok"},
    {"count", employees.size()},
    {"data", employees}   // nlohmann/json แปลง vector<Employee> เป็น JSON array ให้อัตโนมัติ
};
```

รูปแบบ `{"status": ..., "data": ...}` แบบนี้เป็น **convention ที่นิยมมาก** ใน REST API เพราะ
ทำให้ client (เช่น JavaScript frontend) เขียนโค้ดตรวจสอบผลลัพธ์ได้ง่ายและสม่ำเสมอ ไม่ว่า
endpoint ไหนก็ตรวจสอบ `response.status === "ok"` ก่อนเสมอ

---

## 107.8 ตัวอย่างจริง: จาก Database Query สู่ JSON REST API Response (Step 856)

ตอนนี้เรามีทุกชิ้นส่วนที่จำเป็นแล้ว: การ query ฐานข้อมูล PostgreSQL จาก Part 106 และการแปลง
struct เป็น JSON จากหัวข้อที่แล้ว มาลองประกอบร่างเป็นตัวอย่างที่ใกล้เคียงกับสิ่งที่ REST API
endpoint จริงจะต้องทำ:

```cpp
// 05_db_to_json.cpp - อ่านผลลัพธ์จาก PostgreSQL (Part 106) แล้วแปลงเป็น JSON response
// สำหรับ REST API (รูปแบบเดียวกับที่จะใช้ต่อใน Part 108)
#include <pqxx/pqxx>
#include <nlohmann/json.hpp>
#include <iostream>

using json = nlohmann::json;

struct Employee {
    int id;
    std::string name;
    std::string department;
    double salary;

    NLOHMANN_DEFINE_TYPE_INTRUSIVE(Employee, id, name, department, salary)
};

int main(void) {
    try {
        pqxx::connection conn(
            "host=127.0.0.1 port=5432 dbname=coursedb "
            "user=courseuser password=course_pass123"
        );

        pqxx::work txn(conn);
        pqxx::result rows = txn.exec("SELECT id, name, department, salary FROM employees ORDER BY id;");
        txn.commit();

        // แปลงแต่ละแถวของผลลัพธ์ database ให้เป็น Employee struct แล้วเก็บใน vector
        std::vector<Employee> employees;
        for (const auto& row : rows) {
            Employee e;
            e.id = row["id"].as<int>();
            e.name = row["name"].as<std::string>();
            e.department = row["department"].as<std::string>();
            e.salary = row["salary"].as<double>();
            employees.push_back(e);
        }

        // ห่อผลลัพธ์ในรูปแบบ REST API response มาตรฐาน (status + data + count)
        json response = {
            {"status", "ok"},
            {"count", employees.size()},
            {"data", employees}
        };

        std::cout << "=== HTTP Response Body (Content-Type: application/json) ===\n";
        std::cout << response.dump(2) << '\n';

    } catch (const std::exception& e) {
        json error_response = {
            {"status", "error"},
            {"message", e.what()}
        };
        std::cerr << error_response.dump(2) << '\n';
        return 1;
    }
    return 0;
}
```

```bash
g++ -Wall -Wextra -std=c++17 05_db_to_json.cpp -o 05_db_to_json -lpqxx -lpq
./05_db_to_json
```

ผลลัพธ์จริง (query ฐานข้อมูล `coursedb` ที่ตั้งค่าไว้ใน Part 106 ซึ่งขณะนี้เหลือพนักงาน 2 คน
จากตัวอย่าง CRUD ก่อนหน้า):

```
=== HTTP Response Body (Content-Type: application/json) ===
{
  "count": 2,
  "data": [
    {
      "department": "Engineering",
      "id": 1,
      "name": "Somchai",
      "salary": 48000.0
    },
    {
      "department": "Marketing",
      "id": 2,
      "name": "Suda",
      "salary": 38000.0
    }
  ],
  "status": "ok"
}
```

นี่คือ output ที่ **พร้อมส่งกลับเป็น HTTP response body จริง** — ถ้านำ `response.dump()` ไปใส่
ใน `res.set_content(response.dump(), "application/json")` ของ Crow (ทบทวน Part 102) หรือ
`response.send(Pistache::Http::Code::Ok, response.dump())` ของ Pistache (ทบทวน Part 104) ก็
จะได้ REST API endpoint ที่ดึงข้อมูลจากฐานข้อมูลจริงส่งกลับเป็น JSON ได้ทันที — นี่คือสิ่งที่
**Part 108** จะประกอบร่างทั้งหมดเข้าด้วยกันเป็นโปรเจกต์ REST API CRUD แบบเต็มรูปแบบ

### ทิศทางย้อนกลับ: JSON request body -> struct -> SQL parameter

ในทางกลับกัน เมื่อ client ส่ง `POST` request พร้อม JSON body มาเพื่อสร้างพนักงานใหม่ เรา
สามารถ parse body นั้นเป็น `Employee` struct ได้ทันทีด้วย `.get<Employee>()` แล้วส่งค่าจาก
struct เข้า `exec_params` ของ libpqxx ได้เลย (ยกเว้น `id` ที่ควรปล่อยให้ฐานข้อมูล generate
เอง) — เป็นการเชื่อมโยงความรู้ทั้ง 3 Part (105, 106, 107) เข้าด้วยกันอย่างเป็นวงจรครบ: **HTTP
request (JSON) -> C++ struct -> SQL (parameterized query) -> ฐานข้อมูล -> SQL result ->
C++ struct -> JSON -> HTTP response**

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ใช้ `operator[]` อ่านค่าจาก JSON ที่มาจากภายนอกโดยไม่ตรวจสอบก่อน** — ถ้า JSON ที่ client
   ส่งมาขาด field ที่คาดไว้ `j["field"]` จะไม่ throw แต่จะคืน `null` เงียบๆ ทำให้บั๊กแฝงอยู่
   จนกว่าจะพยายามใช้ค่านั้นต่อ (เช่น `.get<std::string>()` จาก `null` จะ throw
   `type_error` แทน ทำให้ error message ไม่ตรงจุดที่แท้จริงของปัญหา) ควรใช้ `.at()` เสมอเมื่อ
   อ่านค่าจากภายนอก
2. **ไม่ครอบ `json::parse()` ด้วย `try`/`catch`** — เมื่อ client ส่ง JSON ผิด syntax มา (เช่น
   ลืมปิด `}`, มี comma เกิน) `json::parse()` จะ throw `parse_error` ทันที ถ้าไม่จับไว้
   โปรแกรมจะ crash ทั้งที่ควรตอบ HTTP 400 Bad Request กลับไปแทน
3. **ลืมว่า field ที่ระบุใน `NLOHMANN_DEFINE_TYPE_INTRUSIVE` ถือเป็น required ทั้งหมด** — ถ้า
   JSON ที่ parse เข้ามาขาด field ใดไปแม้แต่ตัวเดียว `from_json` (ที่ macro สร้างให้) จะ throw
   `out_of_range` ทันที ถ้าต้องการ field แบบ optional ต้องเขียน `from_json` เองด้วย
   `j.value(...)`
4. **สับสนระหว่าง `dump()` กับการพิมพ์ `json` object ตรงๆ ผ่าน `std::cout`** — `std::cout << j`
   ใช้งานได้เพราะ nlohmann/json overload `operator<<` ให้แล้ว แต่ในโค้ดที่ต้องส่งเป็น string
   ไปเก็บตัวแปรหรือส่งผ่าน network (เช่น `res.set_content(j.dump(), ...)`) ต้องเรียก
   `.dump()` เพื่อให้ได้ `std::string` ที่แท้จริง ไม่ใช่พึ่งพา operator overload เพียงอย่างเดียว
5. **ไม่ตรวจสอบว่าค่าที่ parse มาเป็น array จริงก่อนวนลูป** — ถ้าโครงสร้าง JSON ที่คาดไว้เป็น
   array แต่ client ส่ง object มาแทน การวนลูป `for (const auto& x : j)` จะไม่ throw แต่จะได้
   ผลลัพธ์ที่ไม่ตรงกับที่ตั้งใจ (วนลูปผ่าน value ของแต่ละ key แทน) ควรเช็ค `j.is_array()`
   ก่อนเมื่อไม่มั่นใจโครงสร้างของข้อมูลนำเข้า
6. **แปลง `NUMERIC`/`DECIMAL` จากฐานข้อมูลเป็น floating point โดยไม่ระวังเรื่องความแม่นยำ** —
   อย่างที่เห็นในหัวข้อ 107.8 ค่า `salary` ที่มาจาก PostgreSQL `NUMERIC(10,2)` ถูกแปลงเป็น
   `double` แล้วเก็บใน JSON เป็นตัวเลขทศนิยมมาตรฐาน ซึ่งอาจมีปัญหาความแม่นยำเล็กน้อยสำหรับ
   ค่าเงินจำนวนมากๆ ระบบการเงินระดับ production มักส่งค่าเงินเป็น string หรือหน่วยที่เล็กที่สุด
   (สตางค์/cent) แทนที่จะเป็น floating point ตรงๆ

---

## แบบฝึกหัดท้ายบท

1. เขียนโปรแกรมสร้าง JSON object แทนข้อมูล "ตะกร้าสินค้า" (`cart`) ที่มี `customer_name`,
   `items` (array ของ object ที่มี `product_name` และ `quantity`), และ `total_price` แล้ว
   `dump(2)` ออกมาดู
2. เขียนฟังก์ชัน `parse_login_request(const std::string& body)` ที่ parse JSON string
   รูปแบบ `{"username": "...", "password": "..."}` แล้วคืนค่าเป็น `std::pair<std::string,
   std::string>` — ถ้า JSON ผิด format หรือขาด field ใดไปให้จับ exception แล้ว throw
   `std::invalid_argument` พร้อมข้อความที่อ่านเข้าใจง่ายแทน
3. สร้าง struct `Product` ที่มี `id`, `name`, `price`, และ `in_stock` (bool) แล้วใช้
   `NLOHMANN_DEFINE_TYPE_INTRUSIVE` ทดสอบแปลงไป-กลับระหว่าง `std::vector<Product>` กับ JSON
   array ทั้งสองทิศทาง
4. ดัดแปลงตัวอย่าง `05_db_to_json.cpp` ให้ใช้ฐานข้อมูล SQLite จาก Part 105 แทน PostgreSQL
   (ใช้ตาราง `students` จาก `crud_demo.db`) แล้วห่อผลลัพธ์เป็น JSON response รูปแบบเดียวกัน
5. เขียนฟังก์ชัน `Employee from_create_request(const json& body)` ที่รับ JSON body ของ
   `POST /employees` (ซึ่งไม่มี field `id` เพราะฐานข้อมูลจะ generate เอง) แล้วคืนค่าเป็น
   `Employee` struct ที่มี `id = 0` ชั่วคราว จากนั้นเขียนโค้ดเชื่อมต่อกับ `exec_params` ของ
   Part 106 เพื่อ insert และรับค่า `id` จริงกลับมา
6. ลองใช้ `json::diff()` (หรือเขียนฟังก์ชันเปรียบเทียบเองถ้าเวอร์ชันที่ติดตั้งไม่มี) เพื่อหา
   ความแตกต่างระหว่าง JSON สองอันที่เกือบเหมือนกัน (เช่น ข้อมูลพนักงานก่อน/หลัง update) แล้ว
   อธิบายว่าฟีเจอร์นี้มีประโยชน์อย่างไรในการเขียน audit log ของระบบ

### แนวทางเฉลยข้อ 2

```cpp
// exercise2.cpp
#include <nlohmann/json.hpp>
#include <iostream>
#include <stdexcept>
#include <utility>

using json = nlohmann::json;

static std::pair<std::string, std::string> parse_login_request(const std::string& body) {
    try {
        json j = json::parse(body);
        std::string username = j.at("username").get<std::string>();
        std::string password = j.at("password").get<std::string>();
        return {username, password};
    } catch (const json::exception& e) {
        throw std::invalid_argument(
            std::string("request body ไม่ถูกต้อง: ") + e.what()
        );
    }
}

int main(void) {
    // กรณีถูกต้อง
    try {
        auto [user, pass] = parse_login_request(R"({"username":"somchai","password":"1234"})");
        std::cout << "parse สำเร็จ: username=" << user << " password=" << pass << '\n';
    } catch (const std::invalid_argument& e) {
        std::cout << "เกิดข้อผิดพลาด: " << e.what() << '\n';
    }

    // กรณี JSON ผิด syntax
    try {
        auto [user, pass] = parse_login_request(R"({"username":"somchai",)");
        std::cout << "parse สำเร็จ: username=" << user << " password=" << pass << '\n';
    } catch (const std::invalid_argument& e) {
        std::cout << "เกิดข้อผิดพลาด: " << e.what() << '\n';
    }

    // กรณีขาด field password
    try {
        auto [user, pass] = parse_login_request(R"({"username":"somchai"})");
        std::cout << "parse สำเร็จ: username=" << user << " password=" << pass << '\n';
    } catch (const std::invalid_argument& e) {
        std::cout << "เกิดข้อผิดพลาด: " << e.what() << '\n';
    }

    return 0;
}
```

ผลลัพธ์ที่คาดหวังเมื่อคอมไพล์และรัน (`g++ -Wall -Wextra -std=c++17 exercise2.cpp -o exercise2`):

```
parse สำเร็จ: username=somchai password=1234
เกิดข้อผิดพลาด: request body ไม่ถูกต้อง: [json.exception.parse_error.101] parse error at line 1, column 22: syntax error while parsing object key - unexpected end of input; expected string literal
เกิดข้อผิดพลาด: request body ไม่ถูกต้อง: [json.exception.parse_error.101] parse error at line 1, column 22: syntax error while parsing value - unexpected end of input
เกิดข้อผิดพลาด: request body ไม่ถูกต้อง: [json.exception.out_of_range.403] key 'password' not found
```

(หมายเหตุ: ข้อความ error ที่แสดงมาจากการรันจริงบนเครื่องที่ใช้เขียนบทเรียนนี้ ตัวเลขบรรทัด/
คอลัมน์อาจต่างกันเล็กน้อยตามเวอร์ชันของ nlohmann/json แต่โครงสร้างของ exception จะเหมือนกัน)

### แนวทางเฉลยข้อ 3

```cpp
// exercise3.cpp
#include <nlohmann/json.hpp>
#include <iostream>
#include <vector>

using json = nlohmann::json;

struct Product {
    int id;
    std::string name;
    double price;
    bool in_stock;

    NLOHMANN_DEFINE_TYPE_INTRUSIVE(Product, id, name, price, in_stock)
};

int main(void) {
    std::vector<Product> products = {
        {1, "Keyboard", 890.0, true},
        {2, "Mouse", 450.0, true},
        {3, "Webcam", 1200.0, false}
    };

    // vector<Product> -> JSON array
    json j = products;
    std::cout << "vector<Product> -> JSON:\n" << j.dump(2) << "\n\n";

    // JSON array -> vector<Product>
    std::vector<Product> parsed = j.get<std::vector<Product>>();
    std::cout << "แปลงกลับสำเร็จ " << parsed.size() << " รายการ:\n";
    for (const auto& p : parsed) {
        std::cout << "  [" << p.id << "] " << p.name << " ราคา " << p.price
                   << " บาท มีสินค้า=" << std::boolalpha << p.in_stock << '\n';
    }

    return 0;
}
```

ผลลัพธ์ที่คาดหวัง:

```
vector<Product> -> JSON:
[
  {
    "id": 1,
    "in_stock": true,
    "name": "Keyboard",
    "price": 890.0
  },
  {
    "id": 2,
    "in_stock": true,
    "name": "Mouse",
    "price": 450.0
  },
  {
    "id": 3,
    "in_stock": false,
    "name": "Webcam",
    "price": 1200.0
  }
]

แปลงกลับสำเร็จ 3 รายการ:
  [1] Keyboard ราคา 890 บาท มีสินค้า=true
  [2] Mouse ราคา 450 บาท มีสินค้า=true
  [3] Webcam ราคา 1200 บาท มีสินค้า=false
```

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่าทำไม JSON ถึงเป็นมาตรฐานการแลกเปลี่ยนข้อมูลของ Web API และทำไม C++ ต้องพึ่งไลบรารี
  ภายนอกอย่าง **nlohmann/json** เพื่อจัดการมัน
- สร้าง JSON object/array ด้วยมือ, parse JSON string เข้ามาเป็น object, และเข้าถึงค่าด้วย
  `operator[]`, `.at()`, `.value()` พร้อมเข้าใจข้อแตกต่างเชิงความปลอดภัยของแต่ละวิธี
- จัดการ error จาก JSON ผิดรูปแบบผ่าน exception hierarchy (`parse_error`, `out_of_range`,
  `type_error`) ได้อย่างถูกต้องและปลอดภัย
- แปลง C++ struct เป็น/จาก JSON อัตโนมัติด้วย `NLOHMANN_DEFINE_TYPE_INTRUSIVE` โดยไม่ต้องเขียน
  โค้ดแปลงเองเลย รวมถึงใช้งานร่วมกับ `std::vector` ได้ทันที
- ประกอบร่างความรู้จาก Part 105 (SQLite), Part 106 (PostgreSQL) และ Part นี้เข้าด้วยกัน โดย
  แปลงผลลัพธ์จาก database query จริงให้กลายเป็น JSON REST API response ที่พร้อมใช้งาน

ตอนนี้เรามีชิ้นส่วนครบทุกชิ้นแล้ว: HTTP Server (Part 102–104), การเชื่อมต่อฐานข้อมูล
(Part 105–106), และการจัดการ JSON (Part 107) ใน **Part 108** เราจะนำทุกอย่างมาประกอบร่างเป็น
**โปรเจกต์ REST API CRUD ครบวงจร** ตัวจริง ที่รับ request, ตรวจสอบข้อมูล, คุยกับฐานข้อมูล และ
ตอบกลับเป็น JSON ได้อย่างสมบูรณ์แบบ production-ready

**ต่อไป:** [Part 108 — โปรเจกต์: REST API CRUD ครบวงจร](./part-108-rest-api-crud-project.md)
