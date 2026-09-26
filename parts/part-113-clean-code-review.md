# Part 113: Clean Code และ Code Review Practice สำหรับ C/C++ (Step 897–904)

> Module J — Professional และ World-Class Practices | Part 113 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 897–904
> Part ก่อนหน้า: [Part 112 — Deploy: Docker, Nginx, Production Server](./part-112-deploy-docker-nginx.md) | Part ถัดไป: [Part 114 — Software Architecture สำหรับระบบ C++ ขนาดใหญ่](./part-114-software-architecture.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Clean Code** คืออะไร แตกต่างจาก "โค้ดที่ compile ผ่านและทำงานถูกต้อง" อย่างไร
   และทำไมมันถึงเป็นทักษะที่แยกวิศวกร junior ออกจาก senior ได้ชัดเจนที่สุดทักษะหนึ่ง
2. ตั้งชื่อตัวแปร ฟังก์ชัน คลาส และค่าคงที่ ใน C/C++ ได้ถูกต้องตามหลักการที่อ่านแล้วเข้าใจ
   เจตนาทันทีโดยไม่ต้องเดา
3. เขียนฟังก์ชันที่ "สั้นและทำสิ่งเดียว" (Single Responsibility ระดับฟังก์ชัน) ด้วยการแยก
   ฟังก์ชันใหญ่ที่ทำหลายหน้าที่ปนกันออกเป็นฟังก์ชันย่อยที่แต่ละตัวมีความรับผิดชอบเดียว
4. กำจัด Magic Number ออกจากโค้ดด้วย `constexpr` และ `enum class` พร้อมอธิบายได้ว่าทำไมวิธีนี้
   ปลอดภัยและสื่อความหมายกว่าตัวเลขดิบๆ ที่กระจายอยู่ทั่วไฟล์
5. เขียน Comment ที่อธิบาย **"ทำไม"** (เหตุผลเบื้องหลังการตัดสินใจ) แทนที่จะอธิบาย **"อะไร"**
   (สิ่งที่โค้ดเองบอกอยู่แล้ว) และรู้ว่าเมื่อไหร่ไม่ควรมี Comment เลยดีกว่ามี
6. ทำ **Code Review** อย่างมีประสิทธิภาพด้วย Checklist ที่ครอบคลุม และให้ **Feedback ที่สร้างสรรค์**
   ที่โฟกัสที่โค้ด ไม่ใช่ตัวบุคคล
7. Refactor โค้ดจริงที่มีปัญหา Clean Code หลายจุดพร้อมกัน (ชื่อแย่ ฟังก์ชันทำหลายหน้าที่
   magic number comment ไร้ประโยชน์) ให้กลายเป็นโค้ดคุณภาพสูงได้ด้วยตัวเอง
8. เขียน **Commit Message** ที่ดี และรู้ว่า **Pull Request** ที่ดีควรมีองค์ประกอบอะไรบ้าง
   ก่อนขอให้เพื่อนร่วมทีม review

> **หมายเหตุเรื่องสภาพแวดล้อมของ Part นี้**: ทุกตัวอย่างโค้ด C++ ใน Part นี้ถูกคอมไพล์และรันจริง
> บนเครื่องนี้ด้วย `g++ 13.3.0 -Wall -Wextra -Wpedantic -std=c++17` และไม่มี warning ใดๆ เหลืออยู่
> เลย (ยกเว้นตัวอย่างที่ตั้งใจโชว์โค้ดแย่เพื่อการเปรียบเทียบ ซึ่งจะกำกับไว้ชัดเจนว่าเป็น "ตัวอย่างที่
> ไม่ควรทำ") ผลลัพธ์ของโปรแกรมที่แสดงในบทเรียนคือ output จริงที่ได้จากการรันคำสั่งเหล่านั้น

---

## 113.1 Clean Code คืออะไร และทำไมสำคัญกว่าที่คิด (Step 897)

### นิยาม

**Clean Code** คือโค้ดที่**คนอื่นอ่านแล้วเข้าใจเจตนาได้ทันที** โดยใช้ความพยายามน้อยที่สุด — ไม่ใช่
แค่โค้ดที่ compile ผ่านหรือทำงานถูกต้องตาม test case เท่านั้น Robert C. Martin (รู้จักกันในชื่อ
"Uncle Bob") ผู้เขียนหนังสือ *Clean Code* (2008) ให้นิยามสั้นๆ ที่วิศวกรทั่วโลกยึดถือมาจนถึงปัจจุบันว่า:

> **"Clean code always looks like it was written by someone who cares."**
> (โค้ดที่สะอาดมักดูเหมือนถูกเขียนโดยคนที่ใส่ใจมันจริงๆ)

### ทำไม "โค้ดที่ทำงานได้" ยังไม่พอ

ตลอดหลักสูตรนี้ตั้งแต่ Part 1 เราเน้นย้ำเรื่อง `-Wall -Wextra -Wpedantic` และการทดสอบให้โปรแกรม
ทำงานถูกต้อง (Part 93 Unit Testing, Part 95 Static Analysis) แต่ในโลกการทำงานจริง มีความจริงข้อ
หนึ่งที่มือใหม่มักไม่รู้จนกว่าจะเข้าทำงานจริง:

> **โค้ดถูกอ่านมากกว่าถูกเขียนหลายสิบเท่า** (Code is read far more often than it is written)

โปรแกรมเมอร์คนหนึ่งอาจเขียนฟังก์ชันหนึ่งใช้เวลา 10 นาที แต่ตลอดอายุของโปรเจกต์ ฟังก์ชันนั้นอาจถูก
เพื่อนร่วมทีมคนอื่น (หรือตัวเองในอีก 6 เดือนข้างหน้า) เปิดอ่านซ้ำนับสิบนับร้อยครั้งเพื่อ debug,
ต่อยอด, หรือทำความเข้าใจก่อนแก้ไข ถ้าโค้ดอ่านยาก ทุกครั้งที่มีคนต้องอ่านมันคือการเสียเวลาและความ
เสี่ยงที่จะเข้าใจผิดแล้วแก้โค้ดผิดจุด — ต้นทุนสะสมของ "โค้ดอ่านยาก" จึงสูงกว่าต้นทุนตอนเขียนครั้งแรก
มหาศาล

### เปรียบเทียบ: โค้ดที่ "ทำงานได้" กับโค้ดที่ "สะอาด"

| มิติ | โค้ดที่ทำงานได้อย่างเดียว | Clean Code |
|---|---|---|
| Compile ผ่าน `-Wall -Wextra` | ใช่ | ใช่ |
| ผลลัพธ์ถูกต้องตาม requirement | ใช่ | ใช่ |
| ผ่าน Unit Test | ใช่ | ใช่ |
| ชื่อตัวแปร/ฟังก์ชันสื่อเจตนา | อาจจะไม่ | ใช่ |
| แต่ละฟังก์ชันทำหน้าที่เดียว | อาจจะไม่ | ใช่ |
| แก้ไข/ต่อยอดในอนาคตทำได้ง่าย | มักยาก | ง่าย |
| เพื่อนร่วมทีมอ่านแล้วเข้าใจใน 1 นาที | มักไม่ | ใช่ |
| ความเสี่ยงเกิดบั๊กเมื่อมีคนแก้ไขต่อ | สูง | ต่ำ |

จุดสำคัญที่ต้องเข้าใจคือ **Clean Code ไม่ใช่เรื่องความสวยงามผิวเผิน** แต่คือปัจจัยที่ส่งผลโดยตรงต่อ
**ต้นทุนและความเร็วในการพัฒนาซอฟต์แวร์ระยะยาว** ทีมที่มีวินัยเรื่อง Clean Code จะสามารถเพิ่มฟีเจอร์
ใหม่หรือแก้บั๊กได้เร็วขึ้นเรื่อยๆ ในขณะที่ทีมที่ปล่อยให้โค้ดสกปรกสะสม (เรียกว่า **Technical Debt** ซึ่ง
จะพูดถึงลึกใน Part 114) จะพบว่าการเพิ่มฟีเจอร์ใหม่แต่ละครั้งช้าลงเรื่อยๆ จนถึงจุดที่แทบขยับอะไรไม่ได้
เลยโดยไม่ทำของเดิมพัง

### Clean Code สำหรับ C/C++ โดยเฉพาะ ต่างจากภาษาอื่นอย่างไร

หลักการ Clean Code ส่วนใหญ่เป็นสากล (ใช้ได้กับทุกภาษา) แต่ C/C++ มีมิติเพิ่มเติมที่ต้องระวังเป็น
พิเศษ เพราะภาษาให้ "อิสระ" ในการเขียนโค้ดที่อันตรายได้มากกว่าภาษาที่มี Garbage Collector:

| ประเด็น | เหตุผลที่สำคัญเป็นพิเศษใน C/C++ |
|---|---|
| การตั้งชื่อที่บอกความเป็นเจ้าของทรัพยากร | เพราะไม่มี GC ชื่อตัวแปรที่บอกชัดว่าใครเป็นเจ้าของ pointer/handle ช่วยลด double-free และ memory leak |
| Comment อธิบาย Undefined Behavior ที่หลีกเลี่ยงไว้ | โค้ด C/C++ มักมีจุดที่ต้องระวัง UB (Part 5, Part 34) ซึ่งเป็นเรื่องที่ภาษาอื่นไม่ต้องกังวล การอธิบายว่า "ทำไมโค้ดตรงนี้ต้องเขียนแบบนี้เพื่อเลี่ยง UB" มีค่ามาก |
| ฟังก์ชันสั้นช่วยลด scope ของ raw pointer | ฟังก์ชันที่สั้นและทำสิ่งเดียวทำให้ raw pointer (ถ้าจำเป็นต้องมี) มี scope แคบ ลดโอกาส use-after-free |
| Magic number ที่เกี่ยวกับขนาด buffer/memory | ตัวเลขอย่าง `256`, `1024` ที่เกี่ยวกับขนาด buffer เสี่ยงอันตรายกว่าภาษาอื่นมาก เพราะการคำนวณผิดนำไปสู่ buffer overflow (จะเจาะลึกด้าน security ใน Part 115) โดยตรง |

Part นี้จะนำหลักการ Clean Code สากลมาปรับใช้กับบริบทของ C/C++ โดยเฉพาะ พร้อมอ้างอิงกลับไปยัง
Module F (Modern C++ Best Practices, Part 80) ที่เราเรียนเรื่องการเลือกใช้ฟีเจอร์ภาษาไปแล้ว — Part 80
ตอบคำถามว่า **"ควรใช้ฟีเจอร์ไหนของภาษา"** ส่วน Part นี้ตอบคำถามที่กว้างกว่านั้นว่า
**"เขียนโค้ดให้คนอ่านเข้าใจอย่างไร"** ซึ่งเป็นทักษะที่ไม่ผูกกับฟีเจอร์ภาษาตัวใดตัวหนึ่งโดยเฉพาะ

---

## 113.2 การตั้งชื่อ (Naming): ทักษะที่สำคัญที่สุดของ Clean Code (Step 897)

### ทำไมการตั้งชื่อถึงสำคัญที่สุด

Phil Karlton วิศวกรชื่อดังเคยกล่าวไว้ (คำพูดที่ถูกอ้างอิงกันแพร่หลายในวงการซอฟต์แวร์):

> **"There are only two hard things in Computer Science: cache invalidation and naming things."**

ชื่อคือ **Interface แรก** ที่คนอ่านโค้ดเจอ ก่อนจะอ่าน implementation ด้วยซ้ำ ชื่อที่ดีทำให้ผู้อ่าน
"เดาถูก" ว่าฟังก์ชัน/ตัวแปรนั้นทำอะไร โดยไม่ต้องเปิดอ่าน body เลย ชื่อที่แย่ทำตรงกันข้าม — บังคับให้
ผู้อ่านต้องเปิดอ่านทุกบรรทัดของทุกฟังก์ชันที่เกี่ยวข้องเพื่อเข้าใจสิ่งที่ชื่อควรจะบอกไว้แต่แรก

### ตัวอย่าง: ชื่อแย่ vs ชื่อดี

```cpp
// naming_bad.cpp — ตัวอย่างที่ไม่ควรทำ: ชื่อสั้นเกินไป กำกวม ไม่สื่อความหมาย
#include <iostream>
#include <vector>

int f(int a, int b, int c) {
    int x = a * b;
    if (c == 1) {
        x = x + 10;
    }
    return x;
}

int main() {
    std::vector<int> v{1, 2, 3};
    int t = 0;
    for (int i = 0; i < static_cast<int>(v.size()); i++) {
        t = t + v[i];
    }
    std::cout << f(2, 3, 1) << " " << t << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 naming_bad.cpp -o naming_bad && ./naming_bad
```

```
16 6
```

โค้ดนี้ **compile ผ่านไม่มี warning และให้ผลลัพธ์ถูกต้อง** แต่ลองถามตัวเองตรงๆ: อ่านครั้งแรกรู้ไหม
ว่า `f(2, 3, 1)` ทำอะไร? `c == 1` หมายความว่าอะไร? `10` ในนั้นคืออะไร? คำตอบคือไม่มีทางรู้เลย
จนกว่าจะเปิดอ่าน implementation อย่างละเอียด

เทียบกับเวอร์ชันที่ตั้งชื่อดี ทำสิ่งเดียวกันทุกประการ:

```cpp
// naming_good.cpp — ชื่อที่อ่านแล้วรู้เจตนาโดยไม่ต้องอ่าน implementation
#include <iostream>
#include <numeric>
#include <vector>

constexpr int kLoyaltyBonus = 10;

int calculate_order_total(int unit_price, int quantity, bool is_loyalty_member) {
    int subtotal = unit_price * quantity;
    if (is_loyalty_member) {
        subtotal = subtotal + kLoyaltyBonus;
    }
    return subtotal;
}

int main() {
    const std::vector<int> daily_sales{1, 2, 3};
    const int total_sales = std::accumulate(daily_sales.begin(), daily_sales.end(), 0);

    std::cout << calculate_order_total(2, 3, true) << " " << total_sales << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 naming_good.cpp -o naming_good && ./naming_good
```

```
16 6
```

**ผลลัพธ์เหมือนกันทุกประการ** (16 6) แต่เวอร์ชันที่สองอ่านครั้งแรกก็เข้าใจทันทีว่า
`calculate_order_total(2, 3, true)` คือ "คำนวณราคารวมของออเดอร์ที่ราคาต่อหน่วย 2, จำนวน 3 ชิ้น,
ลูกค้าเป็นสมาชิก" — ไม่ต้องเดาอะไรเลย นี่คือความต่างระหว่าง "โค้ดที่ทำงานได้" กับ "โค้ดที่สะอาด"

### กฎการตั้งชื่อที่นำไปใช้ได้ทันที

**1. ชื่อต้องบอกเจตนา ไม่ใช่แค่บอกชนิดข้อมูล**

```cpp
// ไม่ดี — บอกแค่ชนิดข้อมูล ไม่บอกเจตนา
int d;              // d คืออะไร? date? distance? discount?
std::vector<int> v; // v เก็บอะไร?

// ดี — บอกเจตนาชัดเจน
int days_since_last_login;
std::vector<int> customer_ids;
```

**2. หลีกเลี่ยง Encoding แบบเก่า (Hungarian Notation) ใน C++ สมัยใหม่**

ภาษา C ยุคเก่า (และ Windows API แบบเดิม) นิยมใส่ตัวย่อชนิดข้อมูลนำหน้าชื่อ เช่น `iCount` (int),
`szName` (zero-terminated string), `pData` (pointer) — วิธีนี้เคยมีประโยชน์ในยุคที่ IDE ยังไม่ฉลาด
พอจะบอกชนิดข้อมูลให้ดูได้ทันที แต่ปัจจุบัน **ไม่แนะนำแล้ว** เพราะ:

- IDE สมัยใหม่ (Part 1, Part 91) แสดงชนิดข้อมูลให้ดูได้ทันทีอยู่แล้วเมื่อเอาเมาส์ชี้
- ถ้าเปลี่ยนชนิดข้อมูลภายหลัง (เช่น `int` เป็น `int64_t`) ต้องไปแก้ชื่อตัวแปรทุกที่ที่ใช้ ทั้งที่
  ไม่จำเป็น
- Type System ของ C++ (โดยเฉพาะกับ `auto`, Part 69) ทำให้การพึ่งชื่อบอกชนิดข้อมูลไม่สอดคล้องกับ
  แนวทางการเขียน Modern C++ อีกต่อไป

```cpp
// ไม่แนะนำ (สไตล์เก่า)
int iRetryCount = 0;
char* pUserName = nullptr;

// แนะนำ (สไตล์ปัจจุบัน)
int retry_count = 0;
std::string user_name;
```

**3. ชื่อฟังก์ชันควรเป็นคำกริยา ชื่อตัวแปร/คลาสควรเป็นคำนาม**

```cpp
// ฟังก์ชัน: คำกริยา บอกว่า "ทำอะไร"
bool is_valid_order(const Order& order);
double calculate_total_price(const Order& order);
void send_confirmation_email(const Order& order);

// คลาส/ตัวแปร: คำนาม บอกว่า "คืออะไร"
class OrderValidator { /* ... */ };
Order pending_order;
```

**4. ตั้งชื่อ Boolean ให้อ่านแล้วเหมือนประโยคคำถาม**

```cpp
// ไม่ดี — อ่านแล้วไม่รู้ true คือ "มี" หรือ "ไม่มี"
bool status;
bool flag;

// ดี — อ่านโค้ดที่ใช้งานแล้วเหมือนอ่านประโยคภาษาอังกฤษ
bool is_ready;
bool has_permission;
if (is_ready && has_permission) { /* ... */ }
```

**5. ความยาวชื่อควรสัมพันธ์กับ Scope**

ตัวแปรที่มี scope แคบมาก (เช่น loop counter ใน for-loop สั้นๆ) ใช้ชื่อสั้นได้ (`i`, `j`) เพราะบริบท
รอบข้างชัดเจนอยู่แล้ว แต่ตัวแปรที่มี scope กว้าง (member variable ของ class, global constant)
**ควรมีชื่อยาวและชัดเจนกว่าเสมอ** เพราะผู้อ่านอาจเจอมันในที่ที่ห่างไกลจากจุดประกาศมาก

```cpp
// ยอมรับได้ — scope แคบมาก บริบทชัดเจนจาก for-loop
for (int i = 0; i < 10; ++i) {
    std::cout << i << "\n";
}

// ไม่ควรทำ — scope กว้าง (member variable) แต่ชื่อสั้นเกินไป
class Order {
    int q;       // เดาไม่ออกว่า q คืออะไรถ้าเจอในฟังก์ชันอื่นที่ไกลจากจุดประกาศ
};

// ควรทำแทน
class Order {
    int quantity;
};
```

### ตารางสรุปกฎการตั้งชื่อ

| หลักการ | ตัวอย่างไม่ดี | ตัวอย่างดี |
|---|---|---|
| บอกเจตนา ไม่ใช่แค่ชนิดข้อมูล | `int d;` | `int days_since_last_login;` |
| หลีกเลี่ยง Hungarian Notation | `int iCount;` | `int count;` |
| ฟังก์ชันเป็นคำกริยา | `Order o();` | `Order create_order();` |
| Boolean อ่านเป็นคำถามได้ | `bool flag;` | `bool is_ready;` |
| ความยาวสัมพันธ์กับ Scope | `int q;` (member) | `int quantity;` (member) |
| หลีกเลี่ยงชื่อกำกวมข้ามความหมาย | `int f(int a, int b, int c)` | `int calculate_order_total(int price, int qty, bool is_member)` |

---

## 113.3 ฟังก์ชันควรสั้นและทำสิ่งเดียว (Step 898)

### หลักการ Single Responsibility ระดับฟังก์ชัน

เราเรียนหลักการ **Single Responsibility Principle (SRP)** ในระดับคลาสไปแล้วบ้างเมื่อพูดถึง OOP
(และจะเจาะลึกเต็มรูปแบบใน Part 114 กับ SOLID Principles) แต่หลักการเดียวกันนี้ใช้ได้กับ **ระดับ
ฟังก์ชัน** เช่นกัน: **ฟังก์ชันหนึ่งควรทำหน้าที่เดียว และทำหน้าที่นั้นให้ดี**

สัญญาณที่บอกว่าฟังก์ชันหนึ่งทำหลายหน้าที่เกินไป:

- ชื่อฟังก์ชันมีคำว่า "และ" (`and`) แฝงอยู่ในความหมาย เช่น "validate**และ**process**และ**print"
- ฟังก์ชันมี comment คั่นเป็นส่วนๆ ด้วยตัวเลข (`// 1. ...`, `// 2. ...`, `// 3. ...`)
- ฟังก์ชันยาวเกิน 1 หน้าจอ (ประมาณ 40-50 บรรทัด) โดยไม่มีเหตุผลที่หลีกเลี่ยงไม่ได้
- ทดสอบฟังก์ชันนี้ด้วย Unit Test (Part 93) ยากมาก เพราะต้อง setup เงื่อนไขหลายอย่างพร้อมกัน
  เพื่อทดสอบแค่ส่วนเล็กๆ ส่วนหนึ่ง

### ตัวอย่าง: ฟังก์ชันที่ทำหลายหน้าที่ (ไม่ควรทำ)

```cpp
// one_thing_bad.cpp — ตัวอย่างที่ไม่ควรทำ
#include <iostream>
#include <string>

struct Order {
    std::string customer_name;
    int quantity;
    double unit_price;
};

// ฟังก์ชันเดียวทำหลายอย่างพร้อมกัน: validate + คำนวณ + format + print
void process_order(const Order& order) {
    // 1. validate
    if (order.customer_name.empty()) {
        std::cout << "Error: ชื่อลูกค้าห้ามว่าง\n";
        return;
    }
    if (order.quantity <= 0) {
        std::cout << "Error: จำนวนต้องมากกว่า 0\n";
        return;
    }
    if (order.unit_price < 0) {
        std::cout << "Error: ราคาต้องไม่ติดลบ\n";
        return;
    }

    // 2. คำนวณราคารวมพร้อมส่วนลด
    double total = order.quantity * order.unit_price;
    if (order.quantity >= 10) {
        total = total * 0.9;
    }

    // 3. format และ print ใบเสร็จ
    std::cout << "===== ใบเสร็จ =====\n";
    std::cout << "ลูกค้า: " << order.customer_name << "\n";
    std::cout << "จำนวน: " << order.quantity << "\n";
    std::cout << "ราคารวม: " << total << " บาท\n";
    std::cout << "====================\n";
}

int main() {
    process_order(Order{"Somchai", 12, 50.0});
    process_order(Order{"", 1, 10.0});
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 one_thing_bad.cpp -o one_thing_bad && ./one_thing_bad
```

```
===== ใบเสร็จ =====
ลูกค้า: Somchai
จำนวน: 12
ราคารวม: 540 บาท
====================
Error: ชื่อลูกค้าห้ามว่าง
```

โค้ดนี้ทำงานถูกต้อง แต่ `process_order` ทำถึง **4 หน้าที่พร้อมกัน**: validate, คำนวณส่วนลด, format
ข้อความ, และ print ปัญหาที่ตามมาเมื่อโปรเจกต์โตขึ้น:

- ถ้าต้องการทดสอบแค่ "logic การคำนวณส่วนลด" ด้วย Unit Test ต้องสร้าง `Order` ที่ผ่าน validate
  ครบทุกเงื่อนไขก่อน ทั้งที่อยากทดสอบแค่การคำนวณ
- ถ้าอนาคตต้องการส่งใบเสร็จเป็น JSON แทนการ print (เช่นทำ REST API เหมือน Part 108) ต้องแกะ
  logic การคำนวณออกจาก logic การ format ซึ่งเป็นงานที่เสี่ยงทำของเดิมพังถ้าไม่ได้เขียนแยกไว้แต่แรก
- อ่านฟังก์ชันครั้งแรกต้องไล่อ่านทั้ง 4 ส่วนพร้อมกันเพื่อเข้าใจภาพรวม ทั้งที่ถ้าแยกฟังก์ชัน สามารถ
  อ่านแค่ชื่อฟังก์ชันย่อยแล้วเข้าใจ flow ได้ทันทีโดยไม่ต้องเจาะรายละเอียด

### เวอร์ชันที่ถูกต้อง: แยกฟังก์ชันตามหน้าที่

```cpp
// one_thing_good.cpp
#include <iostream>
#include <string>

struct Order {
    std::string customer_name;
    int quantity;
    double unit_price;
};

constexpr int kBulkDiscountThreshold = 10;
constexpr double kBulkDiscountRate = 0.9;

// แต่ละฟังก์ชันทำ "สิ่งเดียว" และมีชื่อบอกเจตนาชัดเจน

bool is_valid_order(const Order& order, std::string& error_message) {
    if (order.customer_name.empty()) {
        error_message = "ชื่อลูกค้าห้ามว่าง";
        return false;
    }
    if (order.quantity <= 0) {
        error_message = "จำนวนต้องมากกว่า 0";
        return false;
    }
    if (order.unit_price < 0) {
        error_message = "ราคาต้องไม่ติดลบ";
        return false;
    }
    return true;
}

double calculate_total_with_discount(const Order& order) {
    double total = order.quantity * order.unit_price;
    if (order.quantity >= kBulkDiscountThreshold) {
        total *= kBulkDiscountRate;
    }
    return total;
}

void print_receipt(const Order& order, double total) {
    std::cout << "===== ใบเสร็จ =====\n";
    std::cout << "ลูกค้า: " << order.customer_name << "\n";
    std::cout << "จำนวน: " << order.quantity << "\n";
    std::cout << "ราคารวม: " << total << " บาท\n";
    std::cout << "====================\n";
}

void process_order(const Order& order) {
    std::string error_message;
    if (!is_valid_order(order, error_message)) {
        std::cout << "Error: " << error_message << "\n";
        return;
    }
    const double total = calculate_total_with_discount(order);
    print_receipt(order, total);
}

int main() {
    process_order(Order{"Somchai", 12, 50.0});
    process_order(Order{"", 1, 10.0});
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 one_thing_good.cpp -o one_thing_good && ./one_thing_good
```

```
===== ใบเสร็จ =====
ลูกค้า: Somchai
จำนวน: 12
ราคารวม: 540 บาท
====================
Error: ชื่อลูกค้าห้ามว่าง
```

**ผลลัพธ์เหมือนเดิมทุกประการ** แต่ตอนนี้:

- `is_valid_order` ทดสอบด้วย Unit Test ได้อิสระ โดยไม่ต้องแตะเรื่องการคำนวณหรือการ print เลย
- `calculate_total_with_discount` เป็น **pure function** (ไม่มี side effect รับ input คืน output
  ตรงไปตรงมา) ทดสอบง่ายที่สุดในบรรดาทั้งหมด
- `process_order` อ่านแล้วเข้าใจ **flow ระดับสูง** ได้ทันทีใน 5 บรรทัด โดยไม่ต้องรู้รายละเอียด
  ว่า validate อย่างไร คำนวณอย่างไร print รูปแบบไหน — นี่คือฟังก์ชันที่ทำหน้าที่ **ประสานงาน**
  (orchestration) เพียงอย่างเดียว ซึ่งเป็นรูปแบบที่อ่านง่ายที่สุด

### กฎง่ายๆ ที่ใช้ตัดสินใจได้ทันที

> **ถ้าอธิบายว่าฟังก์ชันทำอะไรแล้วต้องใช้คำว่า "และ" (and) ในประโยคเดียว ฟังก์ชันนั้นทำเกิน 1 หน้าที่**

ตัวอย่าง: "ฟังก์ชันนี้ validate ออเดอร์**และ**คำนวณราคา**และ**พิมพ์ใบเสร็จ" — มีคำว่า "และ" ถึง 2 ครั้ง
บอกชัดเจนว่าควรแยกเป็นอย่างน้อย 3 ฟังก์ชัน

---

## 113.4 กำจัด Magic Number ด้วย `constexpr` และ `enum class` (Step 899)

### Magic Number คืออะไร ทำไมอันตราย

**Magic Number** คือค่าคงที่ตัวเลข (หรือบางครั้งเป็น string) ที่ถูกเขียนดิบๆ ลงในโค้ดโดยตรง
โดยไม่มีชื่ออธิบายว่ามันคืออะไรหรือทำไมถึงเป็นค่านั้น

```cpp
// magic_bad.cpp — ตัวอย่างที่ไม่ควรทำ
#include <iostream>

int get_http_action(int status) {
    if (status == 200) {
        return 1;
    } else if (status == 404) {
        return 3;
    } else if (status == 500) {
        return 7;
    }
    return 0;
}

int main() {
    std::cout << get_http_action(404) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 magic_bad.cpp -o magic_bad && ./magic_bad
```

```
3
```

ปัญหาของโค้ดนี้ชัดเจนมากเมื่ออ่านออกเสียงดังๆ: "ถ้า status เท่ากับ 200 ให้คืนค่า 1 ถ้าเท่ากับ 404
ให้คืนค่า 3 ถ้าเท่ากับ 500 ให้คืนค่า 7" — ผู้อ่านต้อง**จำเอง**ว่า 200 คือ HTTP OK (จากความรู้ทั่วไป
เรื่อง HTTP), 404 คือ Not Found, 500 คือ Internal Server Error (พอเดาได้จากความรู้ HTTP) **แต่**
ตัวเลขผลลัพธ์ `1`, `3`, `7` **ไม่มีความหมายอะไรเลยที่เดาได้จากภายนอก** ต้องเปิดอ่านทุกจุดที่เรียกใช้
ฟังก์ชันนี้เพื่อรู้ว่า caller ตีความเลข 1, 3, 7 อย่างไร

### เวอร์ชันที่ถูกต้อง: `constexpr` + `enum class`

```cpp
// magic_good.cpp
#include <iostream>

// ---------- ค่าคงที่ของ HTTP status code ที่ใช้ในระบบ ----------
namespace http_status {
constexpr int kOk = 200;
constexpr int kNotFound = 404;
constexpr int kInternalServerError = 500;
}  // namespace http_status

// ---------- enum class บอกความหมายของ "action" อย่างชัดเจน ----------
enum class RetryAction {
    kNone,        // ไม่ต้องทำอะไร (สำเร็จ)
    kLogAndSkip,  // แค่บันทึก log แล้วข้าม (404 = ไม่มีทรัพยากรนี้จริงๆ)
    kRetryLater   // ลองใหม่ภายหลัง (500 = ปัญหาฝั่งเซิร์ฟเวอร์ อาจแก้เองได้)
};

RetryAction decide_retry_action(int status) {
    if (status == http_status::kOk) {
        return RetryAction::kNone;
    }
    if (status == http_status::kNotFound) {
        return RetryAction::kLogAndSkip;
    }
    if (status == http_status::kInternalServerError) {
        return RetryAction::kRetryLater;
    }
    return RetryAction::kNone;
}

int main() {
    const RetryAction action = decide_retry_action(http_status::kNotFound);
    std::cout << "action = " << static_cast<int>(action) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 magic_good.cpp -o magic_good && ./magic_good
```

```
action = 1
```

ตอนนี้เรียก `decide_retry_action(http_status::kNotFound)` แล้วอ่านออกเสียงได้ตรงตัวว่า "ตัดสินใจ
action ที่ควรทำเมื่อเจอ HTTP status Not Found" — ไม่ต้องเดาความหมายของตัวเลขใดๆ อีกเลย

### ทำไมต้องใช้ `constexpr` แทน `#define`

โค้ดสไตล์ C ดั้งเดิม (Part 6) มักใช้ `#define MAX_SIZE 100` แทนค่าคงที่ แต่ใน C++ สมัยใหม่ ควรใช้
`constexpr` แทนเสมอด้วยเหตุผล:

| ประเด็น | `#define` | `constexpr` |
|---|---|---|
| มีชนิดข้อมูล (Type) หรือไม่ | ไม่มี (แค่ text substitution ของ preprocessor) | มี ตรวจสอบชนิดได้ตอน compile |
| ปรากฏใน Debugger หรือไม่ | ไม่ปรากฏ (ถูกแทนที่ไปแล้วตั้งแต่ก่อน compile) | ปรากฏ ดูค่าผ่าน debugger ได้ (Part 37) |
| อยู่ใน Scope/Namespace ได้หรือไม่ | ไม่ได้ (global เสมอ ชนกับชื่ออื่นได้ง่าย) | ได้ (จำกัด scope ด้วย namespace/class ได้) |
| Compiler ตรวจสอบความถูกต้องได้ลึกแค่ไหน | ตรวจไม่ได้เลยจนกว่าจะถูกแทนที่ในโค้ดจริง | ตรวจเต็มรูปแบบเหมือนตัวแปรปกติ |

นี่คือหลักการเดียวกับที่เราสรุปไว้ใน Part 80 (Modern C++ Best Practices) — `constexpr` เป็นหนึ่งใน
ตัวอย่างที่ชัดเจนของการ "มอบความรับผิดชอบด้านความถูกต้องให้กับ Type System" แทนที่จะพึ่งพา
text substitution ดิบๆ แบบ preprocessor

### เมื่อไหร่ควรใช้ `enum class` แทนกลุ่มค่าคงที่

ใช้ `constexpr` แยกตัวเมื่อค่าคงที่นั้นเป็น**อิสระจากกัน** (เช่น ขนาด buffer, timeout) แต่ใช้
`enum class` เมื่อค่าคงที่เหล่านั้นเป็น **กลุ่มตัวเลือกที่ปิด (closed set) ที่ไม่ควรมีค่าอื่นนอกกลุ่มนี้**
เช่น สถานะ, action, ประเภท — เหตุผลสำคัญคือ `enum class` ทำให้ **compiler ช่วยตรวจสอบ
`switch` ให้ครบทุกกรณีได้** (จะเห็นตัวอย่างชัดเจนในหัวข้อ 113.7)

---

## 113.5 Comment ที่ดี: อธิบาย "ทำไม" ไม่ใช่ "อะไร" (Step 900)

### หลักการสำคัญที่สุดของการเขียน Comment

> **Comment ที่ดีอธิบายสิ่งที่โค้ดบอกไม่ได้ (เหตุผล/บริบท) ไม่ใช่สิ่งที่โค้ดบอกอยู่แล้ว (พฤติกรรม)**

ถ้าตั้งชื่อตัวแปรและฟังก์ชันดีตามหัวข้อ 113.2 แล้ว โค้ดส่วนใหญ่ควร "อธิบายตัวเอง" ได้อยู่แล้วโดยไม่
ต้องมี comment เพิ่ม สิ่งที่ comment ควรทำหน้าที่แทนคือ **อธิบายเหตุผลเบื้องหลังที่โค้ดเองไม่มีทางบอก
ได้** เช่น ทำไมถึงเลือกวิธีนี้ ทำไมค่านี้ถึงเป็นค่านี้ มีข้อจำกัดอะไรที่ต้องรู้ไว้

### ตัวอย่างเปรียบเทียบ

```cpp
// comments.cpp
#include <chrono>
#include <iostream>
#include <thread>

int main() {
    int retry_count = 0;

    // ไม่ดี: อธิบาย "อะไร" ที่อ่านจากโค้ดตรงๆ ได้อยู่แล้ว ไม่มีประโยชน์เพิ่ม
    retry_count = retry_count + 1;  // เพิ่มค่า retry_count ทีละ 1

    // ดี: อธิบาย "ทำไม" ถึงต้องรอ 200ms ตรงนี้ — เหตุผลที่โค้ดเองบอกไม่ได้
    // ตั้งไว้ 200ms เพราะจากการทดสอบจริงกับ upstream service พบว่าใช้เวลา
    // recover เฉลี่ย ~150ms หลัง connection reset การรอสั้นกว่านี้ทำให้ retry
    // ครั้งแรกล้มเหลวซ้ำเกือบทุกครั้ง (อ้างอิง incident-2026-08-14)
    std::this_thread::sleep_for(std::chrono::milliseconds(200));

    std::cout << "retry_count = " << retry_count << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 comments.cpp -o comments && ./comments
```

```
retry_count = 1
```

Comment แรก (`// เพิ่มค่า retry_count ทีละ 1`) เป็น comment ที่ **ไม่มีค่าอะไรเลย** เพราะโค้ด
`retry_count = retry_count + 1;` บอกสิ่งเดียวกันชัดเจนอยู่แล้ว การมี comment แบบนี้เป็นภาระเพิ่ม
(ต้องอัปเดตคู่กับโค้ดตลอด ถ้าลืมอัปเดตจะกลายเป็น comment โกหก) โดยไม่ได้ประโยชน์อะไรตอบแทน

Comment ที่สอง (อธิบายเรื่อง `sleep_for(200ms)`) มีค่ามาก เพราะ **ไม่มีทางที่ใครอ่านโค้ด
`std::chrono::milliseconds(200)` เฉยๆ แล้วจะรู้ได้เองว่าทำไมถึงเลือก 200** ตัวเลขนี้มาจากการทดสอบ
จริงกับระบบภายนอก ซึ่งเป็นความรู้ที่อยู่ "นอกโค้ด" โดยสมบูรณ์ — นี่คือสิ่งที่ Comment มีไว้เพื่อเก็บรักษา

### กฎการเขียน Comment ที่นำไปใช้ได้ทันที

| ควรเขียน Comment เมื่อ | ไม่ควรเขียน Comment เมื่อ |
|---|---|
| อธิบายเหตุผลทางธุรกิจที่ไม่ปรากฏในโค้ด (เช่น "ทำไมต้องเช็คเงื่อนไขนี้ก่อน") | โค้ดอ่านแล้วชัดเจนอยู่แล้วว่าทำอะไร |
| เตือนเรื่อง trade-off หรือข้อจำกัดที่ไม่ชัดเจนจากโค้ด (เช่น "ฟังก์ชันนี้ช้าโดยตั้งใจเพื่อ...") | comment แค่พูดซ้ำสิ่งที่ชื่อตัวแปร/ฟังก์ชันบอกอยู่แล้ว |
| อธิบาย Workaround สำหรับบั๊กของ library ภายนอก พร้อมลิงก์อ้างอิง | ใช้ comment แก้ปัญหาชื่อตัวแปร/ฟังก์ชันที่แย่ (ควรแก้ชื่อแทน) |
| เอกสารระดับ API (`///` หรือ Doxygen) ที่บอก precondition/postcondition ของฟังก์ชัน public | comment โค้ดเก่าที่ไม่ใช้แล้วทิ้งไว้เฉยๆ (ควรลบทิ้ง ใช้ Git history แทน Part 15) |
| อธิบายว่าทำไมถึง**ไม่**ใช้วิธีที่ "ดูเหมือนง่ายกว่า" (ป้องกันคนมาแก้กลับไปใช้วิธีเดิมที่มีปัญหา) | เขียน comment ตลกๆ/บ่นเรื่องส่วนตัวที่ไม่เกี่ยวกับโค้ด |

### หลักการที่ลึกกว่านั้น: "ชื่อที่ดีคือ Comment ที่ดีที่สุด"

```cpp
// ไม่ดี: ต้องมี comment มาช่วยอธิบายเพราะชื่อตัวแปร/ฟังก์ชันแย่
// เช็คว่า x มากกว่า MAX_LOGIN_ATTEMPTS หรือไม่ ถ้าใช่ให้ล็อคบัญชี
if (x > 5) {
    lock(u);
}

// ดี: ชื่อดีจนไม่ต้องมี comment เลย อ่านแล้วเข้าใจทันที
constexpr int kMaxLoginAttempts = 5;
if (failed_attempts > kMaxLoginAttempts) {
    lock_account(user);
}
```

เมื่อชื่อดีพอ comment ที่เคยจำเป็นก็หายไปเอง — **การพยายามตั้งชื่อให้ดีก่อนเสมอ แล้วค่อยเติม comment
เฉพาะจุดที่ชื่อช่วยไม่ได้จริงๆ** คือลำดับความคิดที่ถูกต้องของ Clean Code

---

## 113.6 Code Review อย่างมีประสิทธิภาพ: Checklist สำหรับ Reviewer (Step 901)

### ทำไม Code Review ถึงสำคัญ

**Code Review** คือกระบวนการที่เพื่อนร่วมทีมอ่านและตรวจสอบโค้ดก่อนจะ merge เข้า branch หลัก
(โดยทั่วไปผ่าน **Pull Request**, PR) งานวิจัยและประสบการณ์จากบริษัทซอฟต์แวร์ทั่วโลกยืนยันตรงกันว่า
Code Review ที่ทำอย่างจริงจังช่วย:

- จับบั๊กได้ก่อนที่จะเข้าสู่ production (ถูกกว่าการแก้บั๊กหลัง deploy มาก)
- กระจายความรู้เรื่องโค้ดในทีม (ไม่มีใครเป็น "จุดเดียวที่รู้" ส่วนใดส่วนหนึ่งของระบบ)
- รักษาความสม่ำเสมอของ style และสถาปัตยกรรมทั่วทั้งโปรเจกต์
- เป็นโอกาสสอนงาน (mentorship) ระหว่างวิศวกรในทีมโดยธรรมชาติ

### Checklist สำหรับ Reviewer: ตรวจอะไรบ้าง

Checklist นี้เรียงตามลำดับความสำคัญ — Reviewer ที่ดีควรตรวจจากบนลงล่าง ไม่ใช่ไล่ตรวจสไตล์การจัด
วรรคตอนก่อนตรวจ logic (เพราะสไตล์ควรถูกจับด้วยเครื่องมืออัตโนมัติอย่าง clang-format จาก Part 95
ไม่ใช่ให้มนุษย์มาเสียเวลาตรวจเอง)

**ระดับความถูกต้อง (Correctness) — สำคัญที่สุด**

- [ ] โค้ดทำในสิ่งที่ PR description บอกว่าจะทำจริงหรือไม่?
- [ ] มี Edge case ที่ยังไม่ถูกจัดการหรือไม่ (ค่าว่าง, ค่าติดลบ, จำนวนสูงสุด/ต่ำสุด)?
- [ ] มี Unit Test (Part 93) ครอบคลุม logic ใหม่หรือไม่? Test ที่มีทดสอบ "พฤติกรรม" จริงหรือแค่
      ทดสอบผิวเผิน (เช่น เช็คแค่ว่าไม่ throw exception)?
- [ ] ถ้าเป็นโค้ดที่เกี่ยวกับ Memory/Resource (Part 38, Part 67) มีการจัดการที่ถูกต้องตาม RAII
      หรือไม่? (ตรวจตาม Checklist ของ Part 80 ประกอบ)

**ระดับ Clean Code (เนื้อหา Part นี้)**

- [ ] ชื่อตัวแปร/ฟังก์ชัน/คลาส สื่อเจตนาชัดเจนหรือไม่?
- [ ] มีฟังก์ชันที่ยาวเกินไปหรือทำหลายหน้าที่พร้อมกันหรือไม่?
- [ ] มี Magic Number/String หลงเหลืออยู่หรือไม่?
- [ ] Comment ที่มีอยู่อธิบาย "ทำไม" หรือแค่พูดซ้ำสิ่งที่โค้ดบอกอยู่แล้ว?
- [ ] มีโค้ดซ้ำซ้อน (Duplicate Code) ที่ควรแยกเป็นฟังก์ชันร่วมหรือไม่?

**ระดับสถาปัตยกรรมและการออกแบบ (จะเจาะลึกใน Part 114)**

- [ ] โค้ดใหม่นี้ผูก (couple) กับโมดูลอื่นแน่นเกินจำเป็นหรือไม่?
- [ ] ถ้าเป็น public API/interface มีการเปลี่ยนแปลงที่กระทบโค้ดผู้ใช้เดิม (Breaking Change)
      โดยไม่ได้ตั้งใจหรือไม่?

**ระดับ Security (จะเจาะลึกใน Part 115)**

- [ ] มีจุดที่รับ input จากภายนอก (user, network, file) โดยไม่ตรวจสอบก่อนใช้งานหรือไม่?
- [ ] มีการใช้ฟังก์ชันที่รู้กันว่าอันตราย (`strcpy`, `sprintf`, `gets`) หรือไม่?

**ระดับเครื่องมืออัตโนมัติ (ควรถูกบล็อกโดย CI จาก Part 94 ก่อนถึงมือ Reviewer ด้วยซ้ำ)**

- [ ] ผ่าน CI (build, test, static analysis จาก Part 94-96) ทั้งหมดหรือยัง?
- [ ] Format ตรงตาม `.clang-format` ของโปรเจกต์หรือไม่?

> **หลักการสำคัญ**: สิ่งที่เครื่องมืออัตโนมัติตรวจได้ (format, compiler warning, static analysis)
> **ไม่ควรให้มนุษย์มาเสียเวลาตรวจซ้ำใน Code Review** เพราะเป็นการใช้เวลาของคนที่มีค่าที่สุด (สมองคน)
> ไปทำงานที่เครื่องจักรทำได้ดีกว่าและเร็วกว่า — Reviewer ที่ดีควรโฟกัสเวลาไปที่ Correctness, Clean
> Code, และ Architecture ซึ่งเป็นสิ่งที่เครื่องมือยังตรวจแทนมนุษย์ไม่ได้เต็มรูปแบบ

---

## 113.7 การให้ Feedback อย่างสร้างสรรค์ และการ Refactor จริง (Step 902–903)

### หลักการให้ Feedback ที่ไม่ทำร้ายความรู้สึก

Code Review เป็นกิจกรรมที่ละเอียดอ่อนทางความรู้สึกมาก เพราะเป็นการวิจารณ์งานที่คนอื่นตั้งใจทำ
Feedback ที่เขียนไม่ดีสามารถทำลายความสัมพันธ์ในทีมและทำให้คนกลัวที่จะส่ง PR ได้ ในทางกลับกัน
Feedback ที่ดีช่วยให้ทีมเติบโตไปด้วยกันโดยไม่มีใครรู้สึกแย่

**หลักการที่ 1: วิจารณ์โค้ด ไม่ใช่วิจารณ์คน**

```
❌ ไม่ดี: "ทำไมคุณเขียนฟังก์ชันนี้ยาวขนาดนี้ อ่านไม่รู้เรื่องเลย"
✅ ดี:    "ฟังก์ชันนี้ทำหลายหน้าที่พร้อมกัน (validate + คำนวณ + print) ลองแยกออกเป็น
          3 ฟังก์ชันย่อยดูไหมครับ จะช่วยให้ทดสอบแยกแต่ละส่วนได้ง่ายขึ้นด้วย"
```

ประโยคแรกโจมตีตัวบุคคล ("ทำไมคุณ...") ในขณะที่ประโยคที่สองพูดถึงโค้ดล้วนๆ ("ฟังก์ชันนี้...") และ
เสนอทางแก้ที่เป็นรูปธรรมพร้อมเหตุผล

**หลักการที่ 2: ตั้งคำถามแทนการสั่ง เมื่อไม่แน่ใจ 100%**

```
❌ ไม่ดี: "ต้องใช้ std::unique_ptr ตรงนี้ ไม่งั้นไม่ผ่าน"
✅ ดี:    "ตรงนี้เห็น raw pointer ที่ new แต่ไม่เห็น delete ที่ชัดเจน — เป็นไปได้ไหมว่าจะ
          leak? ลองพิจารณาใช้ std::unique_ptr ดูครับ (อ้างอิง Part 67)"
```

**หลักการที่ 3: ชมสิ่งที่ทำดีด้วย ไม่ใช่มีแต่คำวิจารณ์**

Code Review ที่มีแต่จุดที่ต้องแก้ทำให้ผู้เขียนรู้สึกท้อ การชี้จุดที่ทำได้ดี (แม้เล็กน้อย) ควบคู่ไปกับ
จุดที่ต้องปรับปรุง ช่วยให้บรรยากาศการ Review เป็นบวกและสร้างสรรค์มากขึ้น

**หลักการที่ 4: แยกระดับความสำคัญของ Feedback ให้ชัดเจน**

ทีมมืออาชีพหลายแห่งใช้ prefix บอกระดับความสำคัญของแต่ละ comment เพื่อไม่ให้ผู้เขียนสับสนว่า
comment ไหน "ต้องแก้ก่อน merge" กับ comment ไหนเป็นแค่ "ความเห็นส่วนตัว ไม่แก้ก็ได้":

| Prefix ที่นิยมใช้ | ความหมาย |
|---|---|
| `[blocking]` หรือ `[must-fix]` | ต้องแก้ก่อน merge เด็ดขาด (บั๊ก, security issue, ผิด requirement) |
| `[suggestion]` | ข้อเสนอแนะที่ทำให้ดีขึ้น แต่ไม่ merge-blocking |
| `[nitpick]` หรือ `[nit]` | เรื่องเล็กน้อยมาก (สไตล์ส่วนตัว) ผู้เขียนเลือกทำตามหรือไม่ก็ได้ |
| `[question]` | ถามเพื่อทำความเข้าใจ ไม่ได้บอกว่าผิด |

### Refactor จริง: ตัวอย่างจาก Task API (Part 108)

มาดูตัวอย่างที่รวมปัญหา Clean Code หลายจุดพร้อมกันในโค้ดเดียว — จำลองสถานการณ์ฟังก์ชัน validate
ชื่อ task ที่เขียนขึ้นอย่างรีบร้อนในสไตล์ที่พบได้บ่อยเมื่อเวลาโปรเจกต์กระชั้นชิด (ต่อยอดแนวคิดจาก
Task API ที่เราสร้างไว้ใน Part 108)

**ก่อน Refactor**

```cpp
// task_validate_bad.cpp — ตัวอย่างที่ไม่ควรทำ
// จำลองโค้ดจาก Task API (Part 108) ที่เขียนโดยรีบทำให้เสร็จ
#include <iostream>
#include <string>

int chk(std::string s, int f) {
    // เช็ค s
    if (s.size() == 0) {
        return 0;
    }
    // เช็คความยาว
    if (s.size() > 100) {
        return 0;
    }
    int r = 1;
    if (f == 1) {
        // ทำ r เป็น 2
        r = 2;
    }
    return r;
}

int main() {
    std::cout << chk("ซื้อของเข้าบ้าน", 1) << "\n";
    std::cout << chk("", 0) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 task_validate_bad.cpp -o task_validate_bad
./task_validate_bad
```

```
2
0
```

ลองนึกภาพเป็น Reviewer ที่เจอโค้ดนี้ใน Pull Request — ปัญหาที่พบทันที:

1. **ชื่อฟังก์ชัน `chk`** — ย่อมาจาก "check" แต่ check อะไร? ไม่มีทางรู้จากชื่อเลย
2. **พารามิเตอร์ `s`, `f`** — `s` พอเดาได้ว่าเป็น string แต่ `f` คืออะไร? ทำไม `f == 1` ถึงทำให้
   ผลลัพธ์เป็น `2`?
3. **รับ `std::string s` โดย copy** — ตาม Part 80 (`performance-unnecessary-value-param`) ควรรับ
   เป็น `const std::string&` แทน
4. **Magic Number `100`** — ทำไมความยาวสูงสุดถึงเป็น 100? ไม่มี comment อธิบาย
5. **ค่าคืนกลับเป็น `int` (0, 1, 2)** — ความหมายของแต่ละค่ากำกวมมาก ต้องเดาจากบริบทของ caller
6. **Comment ไร้ประโยชน์** — `// เช็ค s`, `// เช็คความยาว`, `// ทำ r เป็น 2` เป็น comment ที่พูดซ้ำ
   สิ่งที่โค้ดบรรทัดถัดไปบอกอยู่แล้ว ไม่ได้เพิ่มความเข้าใจอะไรเลย

**ตัวอย่าง Feedback ที่ Reviewer ควรเขียน** (ใช้หลักการจากหัวข้อก่อนหน้า):

```
[must-fix] ชื่อฟังก์ชัน chk และพารามิเตอร์ s, f อ่านแล้วเดาเจตนาไม่ออกเลยครับ
ช่วยเปลี่ยนเป็นชื่อที่บอกว่ากำลัง validate อะไร (เช่น validate_task_title) และ
พารามิเตอร์ f ที่จริงคือ flag บอกความสำคัญของ task ใช่ไหมครับ ถ้าใช่ลองใช้
enum class แทน int ดู จะทำให้ผู้เรียกฟังก์ชันไม่ต้องเดาว่า 1 หมายถึงอะไร

[suggestion] เลข 100 ในเงื่อนไขความยาว อยากทราบที่มาว่าทำไมถึงเป็น 100 ครับ
ถ้ามีเหตุผล (เช่น ข้อจำกัดจากฝั่ง database) แนะนำให้ตั้งเป็น constexpr พร้อม
ชื่อที่บอกเหตุผลนั้นด้วย

[nit] comment สามบรรทัด (// เช็ค s, // เช็คความยาว, // ทำ r เป็น 2) พูดซ้ำสิ่งที่
โค้ดบรรทัดถัดไปบอกอยู่แล้วครับ ลบออกได้เลย ถ้าตั้งชื่อฟังก์ชัน/ตัวแปรใหม่ตามที่เสนอ
ด้านบน โค้ดจะอธิบายตัวเองได้โดยไม่ต้องมี comment เหล่านี้แล้ว
```

**หลัง Refactor**

```cpp
// task_validate_good.cpp
#include <iostream>
#include <string>

constexpr std::size_t kMaxTaskTitleLength = 100;

enum class TaskPriority { kNormal, kUrgent };
enum class ValidationResult { kValid, kTitleEmpty, kTitleTooLong };

// ฟังก์ชันเดียว ทำสิ่งเดียว: ตรวจสอบว่าชื่อ task ถูกต้องตามกติกาหรือไม่
ValidationResult validate_task_title(const std::string& title) {
    if (title.empty()) {
        return ValidationResult::kTitleEmpty;
    }
    if (title.size() > kMaxTaskTitleLength) {
        return ValidationResult::kTitleTooLong;
    }
    return ValidationResult::kValid;
}

// แยกออกมาต่างหาก: กำหนดระดับความสำคัญของ task ไม่ปนกับการ validate
TaskPriority determine_priority(bool is_urgent_request) {
    return is_urgent_request ? TaskPriority::kUrgent : TaskPriority::kNormal;
}

std::string to_string(ValidationResult result) {
    switch (result) {
        case ValidationResult::kValid:       return "valid";
        case ValidationResult::kTitleEmpty:  return "title ห้ามว่าง";
        case ValidationResult::kTitleTooLong:
            return "title ต้องไม่เกิน " + std::to_string(kMaxTaskTitleLength) + " ตัวอักษร";
    }
    return "unknown";
}

int main() {
    const auto result1 = validate_task_title("ซื้อของเข้าบ้าน");
    const auto priority1 = determine_priority(true);
    std::cout << to_string(result1) << " priority=" << static_cast<int>(priority1) << "\n";

    const auto result2 = validate_task_title("");
    std::cout << to_string(result2) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 task_validate_good.cpp -o task_validate_good
./task_validate_good
```

```
valid priority=1
title ห้ามว่าง
```

**สรุปการเปลี่ยนแปลง**

| ปัญหาเดิม | วิธีแก้ |
|---|---|
| ชื่อฟังก์ชัน `chk` กำกวม | เปลี่ยนเป็น `validate_task_title` บอกเจตนาชัดเจน |
| พารามิเตอร์ `f` (int) กำกวม | แยกออกเป็นฟังก์ชัน `determine_priority` ต่างหาก ใช้ `enum class TaskPriority` |
| รับ `std::string` โดย copy | เปลี่ยนเป็น `const std::string&` |
| Magic Number `100` | เปลี่ยนเป็น `constexpr kMaxTaskTitleLength` |
| ค่าคืนกลับ `int` กำกวม (0, 1, 2) | เปลี่ยนเป็น `enum class ValidationResult` ที่ `switch` ครบทุก case ได้ |
| Comment พูดซ้ำสิ่งที่โค้ดบอกอยู่แล้ว | ลบทิ้ง เพราะชื่อใหม่อธิบายตัวเองได้แล้ว |
| ฟังก์ชันเดียวทำ 2 หน้าที่ (validate + กำหนด priority) | แยกเป็น 2 ฟังก์ชันอิสระ |

สังเกตว่านี่คือตัวอย่างที่รวมหลักการทั้งหมดของ Part นี้ (113.2–113.5) เข้าด้วยกันในจุดเดียว — Clean
Code ไม่ใช่การทำตามกฎข้อเดียวโดดๆ แต่คือการมองโค้ดแล้วเห็นภาพรวมของปัญหาทั้งหมดพร้อมกัน

---

## 113.8 Commit Message ที่ดี และ Pull Request ที่ดี (Step 904)

### ทำไม Commit Message ถึงสำคัญ

Commit message ที่ดีคือ**เอกสารประวัติศาสตร์ของโปรเจกต์** ที่ทุกคนในทีม (รวมถึงตัวเองในอนาคต)
จะใช้ค้นหาว่า "ทำไมโค้ดตรงนี้ถึงเป็นแบบนี้" ผ่านคำสั่งอย่าง `git log`, `git blame` เมื่อเกิดบั๊กและ
ต้องสืบว่า commit ไหนเป็นต้นเหตุ

### โครงสร้าง Commit Message ที่ดี (Conventional Commits)

รูปแบบที่นิยมใช้กันแพร่หลายในวงการปัจจุบันคือ **Conventional Commits**:

```
<type>(<scope>): <คำอธิบายสั้นๆ ไม่เกิน 50 ตัวอักษร ขึ้นต้นด้วยกริยารูป present tense>

<คำอธิบายรายละเอียด (ทางเลือก) — อธิบาย "ทำไม" ต้องเปลี่ยน ไม่ใช่แค่ "อะไร" ที่เปลี่ยน
เพราะ diff เองก็บอก "อะไร" ได้อยู่แล้ว>

<Footer (ทางเลือก) — เช่น "Fixes #123", "BREAKING CHANGE: ..." >
```

| `type` ที่ใช้บ่อย | ความหมาย |
|---|---|
| `feat` | เพิ่มฟีเจอร์ใหม่ |
| `fix` | แก้บั๊ก |
| `refactor` | ปรับโครงสร้างโค้ดโดยไม่เปลี่ยนพฤติกรรม |
| `test` | เพิ่ม/แก้ไข test |
| `docs` | แก้ไขเอกสาร |
| `perf` | ปรับปรุงประสิทธิภาพ |
| `build`/`ci` | เปลี่ยนแปลง build system หรือ CI/CD (Part 94) |

### ตัวอย่างเปรียบเทียบ

```
❌ ไม่ดี: "fix bug"
❌ ไม่ดี: "update code"
❌ ไม่ดี: "asdf"
❌ ไม่ดี: "แก้ไขหลายจุด รวม refactor validate function กับ เพิ่ม test กับ update readme"
   (รวมหลายเรื่องที่ไม่เกี่ยวข้องกันไว้ใน commit เดียว — ควรแยกเป็นหลาย commit)
```

```
✅ ดี:
refactor(task-api): แยก validate_task_title ออกจาก determine_priority

ฟังก์ชัน chk() เดิมทำสองหน้าที่พร้อมกัน (validate ความยาว title และ
กำหนด priority) ทำให้ทดสอบแยกส่วนไม่ได้ และพารามิเตอร์ int f สื่อ
ความหมายไม่ชัดเจน แยกเป็นสองฟังก์ชันอิสระ พร้อมเปลี่ยน return type
เป็น enum class ValidationResult ที่ switch ครบทุก case ได้

Refs: code review PR #42
```

Commit นี้อ่านแล้วเข้าใจทันทีว่า **เปลี่ยนอะไร** (บรรทัดแรก) และ **ทำไมถึงต้องเปลี่ยน** (ย่อหน้า
อธิบาย) — ถ้าอีก 8 เดือนข้างหน้ามีคนใช้ `git blame` เจอ commit นี้ตอนสืบสาเหตุบั๊ก จะเข้าใจบริบท
ทั้งหมดได้ทันทีโดยไม่ต้องไปตามหาคนเขียนมาถามเอง

### กฎการเขียน Commit Message ที่นำไปใช้ได้ทันที

1. **หนึ่ง Commit ควรทำเรื่องเดียว** (Atomic Commit) — ถ้าอธิบาย commit แล้วต้องใช้คำว่า "และ"
   หลายครั้ง ควรแยกเป็นหลาย commit (เหมือนหลักการฟังก์ชันในหัวข้อ 113.3)
2. **บรรทัดแรกใช้กริยารูปคำสั่ง (Imperative mood)** — เขียน "Add validation" ไม่ใช่ "Added
   validation" หรือ "Adds validation" เพราะเวลาอ่าน `git log` แล้วนึกว่ากำลังอ่านคำสั่งที่ commit
   นี้ "จะทำ" กับ codebase (ธรรมเนียมนี้มาจากการที่ Git เองใช้รูปแบบนี้ในข้อความอัตโนมัติ เช่น
   `git revert` ที่สร้างข้อความ "Revert ..." ให้อัตโนมัติ)
3. **อย่า commit โค้ดที่ยังไม่ compile ผ่านหรือ test ไม่ผ่านเข้า branch หลัก** — ทำให้ `git bisect`
   (เครื่องมือหา commit ที่ทำให้เกิดบั๊กด้วยการแบ่งครึ่งค้นหา) ใช้งานไม่ได้ผลถ้ามี commit ที่พังกลาง
   ประวัติศาสตร์
4. **commit บ่อยๆ ในสาขาของตัวเอง (feature branch) ได้อย่างอิสระ** — วินัยเรื่อง Atomic Commit
   สำคัญที่สุดตอน merge เข้า branch หลัก ไม่ใช่ทุกขั้นตอนย่อยระหว่างพัฒนา

### Pull Request ที่ดีควรมีอะไรบ้าง

Pull Request คือหน่วยงานที่ถูก Review จริง (ไม่ใช่ commit เดี่ยวๆ) PR ที่ดีทำให้ Reviewer ทำงานได้
เร็วและแม่นยำขึ้นมาก:

**1. Title ที่บอกเจตนาชัดเจน** — เหมือนบรรทัดแรกของ Commit Message

**2. Description ที่ตอบคำถามหลัก 3 ข้อ**

```markdown
## What (เปลี่ยนอะไร)
แยกฟังก์ชัน validate_task_title() และ determine_priority() ออกจาก chk() เดิม

## Why (ทำไมต้องเปลี่ยน)
chk() เดิมทำสองหน้าที่ปน ทดสอบยาก และ code review ใน #38 ชี้ว่าชื่อ/พารามิเตอร์
สื่อความหมายไม่ชัดเจน (ดู feedback ที่ถูกอ้างถึง)

## How to test (วิธีตรวจสอบว่าถูกต้อง)
รัน `ctest` ใน build/ — เพิ่ม test case ใหม่ 4 เคส ครอบคลุมทุก ValidationResult
```

**3. ขนาดของ PR ต้องเล็กพอที่จะ Review ได้จริง** — งานวิจัยหลายชิ้นในวงการซอฟต์แวร์พบว่า
คุณภาพของ Code Review ลดลงอย่างมากเมื่อ diff มีขนาดใหญ่เกินไป (Reviewer เริ่ม "เลื่อนผ่านๆ" แทน
ที่จะอ่านจริงจัง) — แนวทางที่ดีคือแบ่งงานใหญ่เป็น PR ย่อยๆ ที่แต่ละอันทำเรื่องเดียวจบในตัว
(สอดคล้องกับหลักการ Atomic Commit)

**4. Checklist ก่อนขอ Review** — หลายทีมใช้ PR Template ที่มี checklist ให้ผู้เขียนติ๊กเองก่อนขอ
review เช่น:

```markdown
- [ ] Compile ผ่าน `-Wall -Wextra -Wpedantic` โดยไม่มี warning
- [ ] Unit Test ผ่านทั้งหมด (`ctest`)
- [ ] Static analysis (clang-tidy, cppcheck จาก Part 95) ผ่าน
- [ ] อัปเดตเอกสาร/comment ที่เกี่ยวข้องแล้ว (ถ้ามี)
- [ ] PR นี้ทำเรื่องเดียว ไม่ปนกับงานอื่นที่ไม่เกี่ยวข้อง
```

**5. เชื่อมโยงกับ Issue/Ticket ที่เกี่ยวข้อง** — ใช้ syntax ของแพลตฟอร์ม (เช่น `Fixes #42` บน
GitHub) เพื่อให้ระบบปิด Issue อัตโนมัติเมื่อ PR ถูก merge และรักษาความเชื่อมโยงระหว่างงานกับโค้ด

> **ข้อสังเกตสำคัญ**: PR ที่ดีคือรูปแบบหนึ่งของ Clean Code เช่นกัน — เป้าหมายเดียวกันคือทำให้คนอื่น
> (ในที่นี้คือ Reviewer) เข้าใจเจตนาของงานได้เร็วที่สุดโดยใช้ความพยายามน้อยที่สุด หลักการที่เราเรียน
> มาทั้งหมดใน Part นี้ (ตั้งชื่อดี, แยกความรับผิดชอบ, อธิบายเหตุผล) ล้วนนำไปใช้ได้กับการสื่อสารใน
> ทีมเช่นเดียวกับที่ใช้กับโค้ด

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **มองว่า Clean Code คือ "ความชอบส่วนตัว" ที่เถียงกันไม่จบ** — หลักการส่วนใหญ่ใน Part นี้ (ชื่อ
   สื่อเจตนา, ฟังก์ชันทำสิ่งเดียว, ไม่มี magic number) เป็นข้อเท็จจริงเชิงวิศวกรรมที่วัดผลได้
   (ทดสอบง่ายขึ้นจริง, บั๊กน้อยลงจริง) ไม่ใช่แค่รสนิยม สิ่งที่เป็นรสนิยมจริงๆ (เช่น จัดวงเล็บบรรทัด
   เดียวกันหรือขึ้นบรรทัดใหม่) ควรถูกกำหนดด้วย `.clang-format` (Part 95) ครั้งเดียวแล้วจบ ไม่ต้อง
   เถียงกันใน Code Review อีก
2. **Refactor ทุกอย่างในคราวเดียวโดยไม่มี Test รองรับ** — เหมือนที่เตือนไว้ใน Part 80 การ
   Refactor ควรทำทีละเล็กละน้อย พร้อม Unit Test ยืนยันว่าพฤติกรรมเดิมไม่เปลี่ยน การ refactor
   ฟังก์ชันใหญ่หลายตัวพร้อมกันในโค้ดที่ไม่มี test คุ้มครองเสี่ยงทำของเดิมพังโดยไม่รู้ตัว
3. **ตั้งชื่อยาวเกินจำเป็นจนอ่านยากกว่าเดิม** — Clean Code ไม่ได้แปลว่า "ชื่อยิ่งยาวยิ่งดี" เช่น
   `theTotalNumberOfItemsCurrentlyInTheShoppingCartRightNow` ยาวเกินจำเป็น ทั้งที่
   `cart_item_count` สื่อความหมายเดียวกันและอ่านง่ายกว่ามาก
4. **เขียน Comment อธิบายโค้ดที่ควรแก้ไขแทน** — เช่น `// ระวัง! ฟังก์ชันนี้บั๊กถ้า x เป็นลบ`
   เป็น comment ที่บอกว่า "รู้อยู่แล้วว่ามีบั๊ก" แต่ไม่แก้ — ควรแก้บั๊กจริง ไม่ใช่แค่เตือนไว้เฉยๆ
5. **ให้ Feedback ใน Code Review แบบ "สั่งการ" โดยไม่อธิบายเหตุผล** — comment สั้นๆ ว่า "ผิด
   ต้องแก้" โดยไม่บอกว่าผิดตรงไหน ทำไมถึงผิด และควรแก้อย่างไร ทำให้ผู้เขียนไม่ได้เรียนรู้อะไรเลย
   แม้จะแก้ตามที่บอกก็ตาม
6. **ปล่อยให้ PR ใหญ่เกินไปจน Reviewer "เลื่อนผ่านๆ" โดยไม่อ่านจริง** — PR ที่มี diff หลายพัน
   บรรทัดมักถูก approve อย่างรวดเร็วโดยไม่มีใครอ่านจริงจัง เพราะขนาดใหญ่เกินกว่าจะย่อยได้ในเวลาจำกัด
   ทำให้ Code Review สูญเสียประโยชน์ที่แท้จริงไปโดยสิ้นเชิง
7. **สับสนระหว่าง `enum` ธรรมดากับ `enum class`** — เมื่อกำจัด magic number ด้วย enum ควรใช้
   `enum class` เสมอตามที่เรียนใน Part 69/80 ไม่ใช่ `enum` ธรรมดา เพราะ `enum` ธรรมดาแปลงเป็น
   `int` อัตโนมัติและรั่วชื่อค่าคงที่เข้า scope ภายนอก ทำให้เผลอเปรียบเทียบ enum คนละกลุ่มกันได้
   โดย compiler ไม่เตือน

---

## แบบฝึกหัดท้ายบท

1. หยิบฟังก์ชัน `chk()` เวอร์ชัน "ก่อน Refactor" ในหัวข้อ 113.7 ลองเขียน Feedback ของตัวเอง
   (ไม่ copy จากบทเรียน) โดยใช้ระดับความสำคัญ `[must-fix]`, `[suggestion]`, `[nit]` ให้ครบทั้ง 3
   ระดับ อย่างน้อยระดับละ 1 ข้อ

2. เขียนฟังก์ชันคำนวณค่าปรับหนังสือคืนช้าของห้องสมุด ที่มีปัญหา Clean Code แบบเดียวกับหัวข้อ
   113.7 (ชื่อสั้น พารามิเตอร์กำกวม magic number) จากนั้น Refactor ให้สะอาดตามหลักการทั้งหมดที่
   เรียนมา (ดูแนวทางเฉลยข้อ 2 ด้านล่าง)

3. หาไฟล์ `.cpp` ที่ตัวเองเคยเขียนใน Part ก่อนหน้าของหลักสูตรนี้ (เลือกมา 1 ไฟล์) แล้วใช้
   Checklist ในหัวข้อ 113.6 ตรวจสอบเอง เขียนรายงานสั้นๆ ว่าพบปัญหาอะไรบ้าง และจะแก้ไขอย่างไร

4. เขียน Commit Message ที่ดีสำหรับสถานการณ์สมมตินี้: "เปลี่ยนฟังก์ชัน `calculate_total()` ที่รับ
   `std::vector<int>` โดย copy ให้รับเป็น `const std::vector<int>&` แทน เพื่อลดการ copy ที่ไม่
   จำเป็นเมื่อ vector มีขนาดใหญ่" ต้องมีทั้งบรรทัดแรก (summary) และย่อหน้าอธิบายเหตุผล

5. เขียน PR Description แบบเต็ม (What/Why/How to test) สำหรับการเปลี่ยนแปลงในข้อ 4 พร้อม
   Checklist ก่อนขอ Review อย่างน้อย 4 ข้อ

6. อธิบายด้วยคำพูดตัวเอง (เขียนเป็นข้อความหรือ comment) ว่าทำไม `enum class` ถึงปลอดภัยกว่า
   `enum` ธรรมดาเมื่อใช้แทน magic number ยกตัวอย่างสถานการณ์ที่ `enum` ธรรมดาทำให้เกิดบั๊กที่
   compiler ไม่เตือน แต่ `enum class` จะป้องกันได้

### แนวทางเฉลยข้อ 1

```
[must-fix] ชื่อฟังก์ชัน chk() และพารามิเตอร์ s, f ไม่สื่อเจตนาเลยครับ ผู้เรียกฟังก์ชันนี้
ต้องเปิดอ่าน implementation ทุกครั้งเพื่อรู้ว่าค่าที่คืนกลับ (0, 1, 2) หมายถึงอะไร
ซึ่งเสี่ยงมากถ้ามีคนเรียกใช้ผิดความหมายในอนาคต แนะนำให้เปลี่ยนเป็น
validate_task_title() ที่คืนค่าเป็น enum class ValidationResult แทน int ดิบๆ

[suggestion] พารามิเตอร์ std::string s รับโดย copy ทั้งที่ไม่มีจุดไหนในฟังก์ชันแก้ไข
ค่าของมันเลย ตาม Part 80 (performance-unnecessary-value-param) แนะนำให้เปลี่ยนเป็น
const std::string& เพื่อลดการ copy ที่ไม่จำเป็น

[nit] comment "// เช็ค s" และ "// เช็คความยาว" พูดซ้ำสิ่งที่โค้ดบรรทัดถัดไปบอกอยู่แล้ว
ลบออกได้เลยครับ ถ้าตั้งชื่อฟังก์ชันใหม่ตาม suggestion ด้านบน โค้ดจะอธิบายตัวเองได้
โดยไม่ต้องมี comment เหล่านี้อีกต่อไป
```

**อธิบาย**: สังเกตว่า feedback ทั้ง 3 ระดับพูดถึง**โค้ด**ล้วนๆ ไม่มีคำใดพาดพิงถึงตัวผู้เขียน และทุกข้อ
เสนอทางแก้ที่เป็นรูปธรรม (ไม่ใช่แค่บอกว่า "ผิด") พร้อมอ้างอิงกลับไปยัง Part ที่เกี่ยวข้องในหลักสูตร
เพื่อให้ผู้เขียนไปอ่านรายละเอียดเพิ่มเติมได้เอง — นี่คือรูปแบบ feedback ที่ทั้งช่วยแก้ปัญหาเฉพาะหน้า
และช่วยให้ผู้เขียนเรียนรู้ไปพร้อมกัน

### แนวทางเฉลยข้อ 2

**ก่อน Refactor** (ตัวอย่างปัญหา Clean Code ที่ตั้งใจสร้างขึ้นสำหรับแบบฝึกหัดนี้):

```cpp
// ex2_bad.cpp — ตัวอย่างที่ไม่ควรทำ
#include <iostream>

// ฟังก์ชันคำนวณค่าปรับหนังสือคืนช้าจากห้องสมุด (สไตล์แย่)
double calc(int d, int t) {
    double p = 0;
    if (t == 0) {
        if (d > 7) {
            p = (d - 7) * 5;
        }
    } else if (t == 1) {
        if (d > 3) {
            p = (d - 3) * 10;
        }
    } else {
        if (d > 14) {
            p = (d - 14) * 2;
        }
    }
    if (p > 200) {
        p = 200;
    }
    return p;
}

int main() {
    std::cout << calc(10, 0) << "\n";
    std::cout << calc(5, 1) << "\n";
    std::cout << calc(100, 2) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex2_bad.cpp -o ex2_bad && ./ex2_bad
```

```
15
20
172
```

**หลัง Refactor**:

```cpp
// ex2_good.cpp
#include <algorithm>
#include <iostream>

// ---------- ค่าคงที่แทน magic number ทุกตัว พร้อมชื่อสื่อความหมาย ----------
enum class BookType { kGeneral, kNewRelease, kReferenceOnly };

namespace loyalty_period {
constexpr int kGeneralGraceDays = 7;
constexpr int kNewReleaseGraceDays = 3;
constexpr int kReferenceGraceDays = 14;
}  // namespace loyalty_period

namespace fine_rate {
constexpr double kGeneralPerDay = 5.0;
constexpr double kNewReleasePerDay = 10.0;
constexpr double kReferencePerDay = 2.0;
}  // namespace fine_rate

constexpr double kMaxFine = 200.0;

// แยกคำนวณ "จำนวนวันที่เกินกำหนดจริง" ออกมาต่างหาก อ่านง่ายกว่ารวมไว้ในที่เดียว
int calculate_overdue_days(int days_late, int grace_period_days) {
    const int overdue = days_late - grace_period_days;
    return overdue > 0 ? overdue : 0;
}

double calculate_late_fee(int days_late, BookType type) {
    double fee = 0.0;
    switch (type) {
        case BookType::kGeneral:
            fee = calculate_overdue_days(days_late, loyalty_period::kGeneralGraceDays)
                  * fine_rate::kGeneralPerDay;
            break;
        case BookType::kNewRelease:
            fee = calculate_overdue_days(days_late, loyalty_period::kNewReleaseGraceDays)
                  * fine_rate::kNewReleasePerDay;
            break;
        case BookType::kReferenceOnly:
            fee = calculate_overdue_days(days_late, loyalty_period::kReferenceGraceDays)
                  * fine_rate::kReferencePerDay;
            break;
    }
    return std::min(fee, kMaxFine);
}

int main() {
    std::cout << calculate_late_fee(10, BookType::kGeneral) << "\n";
    std::cout << calculate_late_fee(5, BookType::kNewRelease) << "\n";
    std::cout << calculate_late_fee(100, BookType::kReferenceOnly) << "\n";
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex2_good.cpp -o ex2_good && ./ex2_good
```

```
15
20
172
```

**อธิบาย**: ผลลัพธ์เหมือนกันทุกประการ (15, 20, 172) ยืนยันว่าการ Refactor ไม่เปลี่ยนพฤติกรรมของ
โปรแกรม แต่เวอร์ชันใหม่แก้ปัญหาทั้งหมดที่พบในเวอร์ชันเดิม: `t == 0/1/else` กลายเป็น
`enum class BookType` ที่ compiler ช่วยตรวจสอบ `switch` ให้ครบทุก case, ตัวเลข `7`, `3`, `14`,
`5.0`, `10.0`, `2.0`, `200` ทั้งหมดมีชื่อสื่อความหมายผ่าน `constexpr` ที่จัดกลุ่มด้วย `namespace`
และ logic การคำนวณ "จำนวนวันเกินกำหนด" ถูกแยกออกมาเป็นฟังก์ชันย่อย `calculate_overdue_days`
ที่ทดสอบแยกได้อิสระจากอัตราค่าปรับ — ถ้าในอนาคตห้องสมุดเปลี่ยนนโยบาย (เช่น เพิ่มประเภทหนังสือใหม่)
compiler จะเตือนทันทีถ้า `switch` ไม่ครอบคลุมกรณีใหม่ ต่างจากเวอร์ชันเดิมที่ใช้ `if-else` ธรรมดาซึ่ง
ไม่มีการเตือนใดๆ เลยถ้าลืมจัดการกรณีใหม่

---

## สรุปท้ายบท

ใน Part นี้ ซึ่งเป็น Part แรกของ **Module J — Professional และ World-Class Practices** เราได้:

- เข้าใจว่า **Clean Code** ต่างจาก "โค้ดที่ทำงานได้" อย่างไร และทำไมมันถึงส่งผลต่อต้นทุนการพัฒนา
  ซอฟต์แวร์ระยะยาวโดยตรง
- เรียนรู้กฎการ **ตั้งชื่อ** ที่นำไปใช้ได้ทันที ทั้งเรื่องการบอกเจตนา หลีกเลี่ยง Hungarian Notation
  และการเลือกความยาวชื่อให้สัมพันธ์กับ Scope
- ฝึกแยกฟังก์ชันที่ทำหลายหน้าที่ให้กลายเป็นฟังก์ชันที่ **ทำสิ่งเดียวและทำได้ดี** พร้อมเห็นผลจริงว่า
  โค้ดที่แยกดีแล้วทดสอบและต่อยอดได้ง่ายกว่ามากแค่ไหน
- กำจัด **Magic Number** ด้วย `constexpr` และ `enum class` เข้าใจว่าทำไมสองเครื่องมือนี้ปลอดภัย
  กว่า `#define` และตัวเลขดิบๆ
- เรียนรู้ว่า **Comment ที่ดี** อธิบาย "ทำไม" ไม่ใช่ "อะไร" และเข้าใจว่าชื่อที่ดีคือ comment ที่ดี
  ที่สุด
- ได้ **Checklist สำหรับ Code Review** ที่ครอบคลุมตั้งแต่ความถูกต้อง Clean Code สถาปัตยกรรม
  ไปจนถึง Security พร้อมหลักการให้ **Feedback อย่างสร้างสรรค์** ที่วิจารณ์โค้ดไม่ใช่ตัวบุคคล
- Refactor โค้ดจริงจาก Task API (Part 108) ที่มีปัญหา Clean Code หลายจุดพร้อมกัน ให้กลายเป็น
  โค้ดคุณภาพสูง พร้อมตัวอย่าง Feedback ที่ Reviewer ควรเขียนจริง
- เรียนรู้การเขียน **Commit Message** ตามรูปแบบ Conventional Commits และองค์ประกอบของ
  **Pull Request** ที่ดีที่ทำให้ Code Review มีประสิทธิภาพสูงสุด

หลักการทั้งหมดใน Part นี้เป็นรากฐานสำคัญที่จะนำไปใช้ตลอด Module J — เพราะไม่ว่าจะออกแบบ
สถาปัตยกรรมที่ดีแค่ไหน (Part 114) หรือเขียนโค้ดที่ปลอดภัยแค่ไหน (Part 115) ถ้าโค้ดนั้นอ่านไม่รู้เรื่อง
คุณค่าของงานวิศวกรรมที่ดีก็จะสื่อสารไปถึงเพื่อนร่วมทีมไม่ได้เต็มที่

ใน **Part 114** เราจะยกระดับจากมุมมอง "ฟังก์ชันเดียว" สู่มุมมอง **"ทั้งระบบ"** ด้วย **Software
Architecture สำหรับระบบ C++ ขนาดใหญ่** เริ่มจาก **SOLID Principles** ที่ต่อยอดจาก Abstract Class
และ Interface ที่เราเรียนไปแล้วใน Part 50 ไปจนถึง Layered Architecture ที่ใช้ REST API จาก
Part 108 เป็นกรณีศึกษาจริง

**ต่อไป:** [Part 114 — Software Architecture สำหรับระบบ C++ ขนาดใหญ่](./part-114-software-architecture.md)
