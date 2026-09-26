# Part 66: Custom Allocator ใน STL (Step 521–528)

> Module E — Templates, Generic Programming และ STL | Part 66 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 521–528
> Part ก่อนหน้า: [Part 65 — std::string และ string_view/Regex ขั้นสูง](./part-065-string-advanced.md) | Part ถัดไป: [Part 67 — Smart Pointer (unique/shared/weak_ptr)](./part-067-smart-pointers.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Allocator** คืออะไร และทำไม STL container ถึงแยกกลไก "จัดสรร/คืน
   หน่วยความจำ" ออกจาก "logic ของ container" อย่างชัดเจน
2. อธิบายการทำงานของ `std::allocator` เริ่มต้นที่ container ทุกตัวใช้ ถ้าไม่ได้ระบุเอง
3. เข้าใจ Allocator interface ขั้นต่ำตามมาตรฐาน C++17 (`value_type`, `allocate`,
   `deallocate`, converting constructor, `operator==`/`operator!=`)
4. ระบุสถานการณ์จริงที่การเขียน custom allocator คุ้มค่า (performance-critical code,
   memory pool, embedded system) และสถานการณ์ที่ไม่คุ้มค่า
5. เขียน pool allocator แบบง่ายๆ ของตัวเองได้ และนำไปใช้กับ `std::vector<int, MyAllocator<int>>`
   ได้จริง
6. เข้าใจแนวคิด `std::allocator_traits` และ rebind สำหรับ container แบบ node-based เช่น
   `std::list`
7. ตระหนักถึงข้อควรระวังและความซับซ้อนของ custom allocator (stateful allocator, การ
   เปรียบเทียบความเท่ากัน, exception safety) และเข้าใจว่าทำไมงานจริงส่วนใหญ่ไม่จำเป็นต้อง
   เขียนเอง แต่ควรเข้าใจกลไกเบื้องหลัง

---

## 66.1 Allocator คืออะไร (Step 521)

ทุกครั้งที่เราเขียน `std::vector<int> v; v.push_back(42);` เบื้องหลังมีการ "ขอหน่วยความจำ"
เกิดขึ้นเพื่อเก็บข้อมูล คำถามคือ: ใครเป็นคนขอหน่วยความจำนั้น? คำตอบคือไม่ใช่ `std::vector`
เองโดยตรง แต่เป็น **Allocator** ที่ `std::vector` "มอบหมาย" งานนี้ให้ทำแทน

STL container ทุกตัวถูกออกแบบตามหลักการ **แยกความรับผิดชอบ (Separation of Concerns)**:

- **Container** (เช่น `std::vector`, `std::list`, `std::map`) รับผิดชอบเรื่อง **logic
  ของโครงสร้างข้อมูล** — จะเก็บ element เรียงกันอย่างไร จะขยายขนาดเมื่อไหร่ จะเชื่อม node
  แบบไหน
- **Allocator** รับผิดชอบเรื่อง **การจัดสรรและคืนหน่วยความจำดิบ (raw memory)** เท่านั้น —
  ไม่รู้อะไรเลยเกี่ยวกับ "logic" ของ container ที่ใช้มันอยู่

ลองดู signature เต็มของ `std::vector` ให้ชัดเจน:

```cpp
#include <memory>

template <
    typename T,
    typename Allocator = std::allocator<T>
>
class vector;
```

พารามิเตอร์ตัวที่สอง `Allocator` มีค่า default เป็น `std::allocator<T>` นี่คือเหตุผลที่
เราเขียน `std::vector<int>` โดยไม่ต้องระบุ allocator ก็ได้ตลอดมา เพราะ compiler เติม
`std::allocator<int>` ให้อัตโนมัติ — แต่ในความเป็นจริงเราสามารถเปลี่ยน allocator ตัวที่สอง
นี้เป็นของเราเองได้ ตราบใดที่มันทำตาม "สัญญา" (interface) ที่ container คาดหวัง

### เปรียบเทียบให้เห็นภาพ

ลองนึกภาพร้านอาหาร (container) กับซัพพลายเออร์วัตถุดิบ (allocator):

- ร้านอาหารรู้แค่ว่า "ต้องการหมู 2 กิโล" (`allocate(n)`) และ "เอาหมูที่เหลือไปคืน"
  (`deallocate(ptr, n)`)
- ร้านอาหารไม่สนใจว่าซัพพลายเออร์จะไปเอาหมูมาจากไหน จากฟาร์มไหน ขนส่งด้วยรถแบบไหน
- ถ้าอยากเปลี่ยนซัพพลายเออร์ (เช่นจากตลาดสดเป็นฟาร์มโดยตรงเพื่อความสดและถูกกว่า)
  ร้านอาหารก็ยังทำอาหารสูตรเดิมได้ทุกประการ เพียงแค่เปลี่ยนแหล่งวัตถุดิบเท่านั้น

นี่คือเหตุผลที่การออกแบบแบบนี้ทรงพลังมาก: เราสามารถเปลี่ยนวิธีจัดสรรหน่วยความจำของ
`std::vector` ได้อย่างสิ้นเชิง (เช่นให้ไปใช้ memory pool ที่เตรียมไว้ล่วงหน้าแทนการเรียก
`malloc`/`new` ของระบบทุกครั้ง) โดย**ไม่ต้องแก้โค้ด logic ของ vector เองแม้แต่บรรทัดเดียว**

---

## 66.2 std::allocator เริ่มต้นทำงานอย่างไร (Step 522)

`std::allocator<T>` (อยู่ใน header `<memory>`) คือ allocator เริ่มต้นที่ container ทุกตัว
ใน STL ใช้ ถ้าไม่ได้ระบุเป็นอย่างอื่น ภายในมันค่อนข้างเรียบง่าย: `allocate(n)` เรียก
`::operator new` เพื่อขอหน่วยความจำดิบขนาด `n * sizeof(T)` ไบต์ ส่วน `deallocate(ptr, n)`
เรียก `::operator delete` คืนหน่วยความจำนั้นกลับไป — พูดง่ายๆ คือเป็น wrapper บางๆ ห่อ
`new`/`delete` ของ C++ ไว้อีกชั้นหนึ่ง

สิ่งสำคัญที่ต้องเข้าใจคือ allocator ไม่ได้ทำแค่ "จองพื้นที่" กับ "คืนพื้นที่" เท่านั้น
แต่ยังแยกขั้นตอนระหว่าง **การจัดสรรหน่วยความจำดิบ** กับ **การสร้าง/ทำลายอ็อบเจกต์** ออก
จากกันอย่างชัดเจน ทั้งสี่ขั้นตอนนี้คือหัวใจของ allocator ทุกตัว:

| ขั้นตอน | หน้าที่ |
|---|---|
| `allocate(n)` | ขอหน่วยความจำ**ดิบ** (raw memory) พอสำหรับ `n` อ็อบเจกต์ — ยังไม่มี constructor ใดถูกเรียก |
| `construct(ptr, args...)` | สร้างอ็อบเจกต์จริงบนหน่วยความจำที่จองไว้ (ใช้ placement new ภายใน) |
| `destroy(ptr)` | เรียก destructor ของอ็อบเจกต์ที่ตำแหน่งนั้น (ยังไม่คืนหน่วยความจำ) |
| `deallocate(ptr, n)` | คืนหน่วยความจำ**ดิบ**กลับให้ระบบ (ต้องเรียก `destroy` ให้ครบก่อนเสมอ) |

การแยกขั้นตอนแบบนี้สำคัญมาก เพราะ container อย่าง `std::vector` ต้องสามารถ "จองที่ไว้
ล่วงหน้า" (`reserve`) โดยยังไม่สร้างอ็อบเจกต์ใดๆ เลยได้ — ถ้า `allocate` สร้างอ็อบเจกต์
ไปพร้อมกันเลย `reserve(1000)` จะต้องเรียก default constructor 1000 ครั้งทันที ซึ่งขัดกับ
พฤติกรรมจริงของ `std::vector` ที่ `reserve` ไม่สร้างอ็อบเจกต์ใดๆ จนกว่าจะมีการ `push_back`
หรือ `emplace_back` จริง

```cpp
#include <iostream>
#include <memory>
#include <string>

int main() {
    // std::allocator<T> คือ allocator เริ่มต้นที่ container ทุกตัวใน STL ใช้ ถ้าเราไม่ระบุเอง
    std::allocator<std::string> alloc;

    // 1) allocate: ขอหน่วยความจำ "ดิบ" สำหรับ 3 อ็อบเจกต์ std::string
    //    หมายเหตุ: allocate แค่ "จอง" หน่วยความจำ ยังไม่ได้เรียก constructor ใดๆ ทั้งสิ้น
    std::string* buffer = alloc.allocate(3);

    // 2) construct: สร้างอ็อบเจกต์จริงลงบนหน่วยความจำที่จองไว้ (placement new ภายใน)
    //    ใน C++17 เรียกผ่าน std::allocator_traits ซึ่งเป็นวิธีมาตรฐานที่ container ใช้จริง
    using Traits = std::allocator_traits<std::allocator<std::string>>;
    Traits::construct(alloc, &buffer[0], "แอปเปิ้ล");
    Traits::construct(alloc, &buffer[1], "กล้วย");
    Traits::construct(alloc, &buffer[2], "ส้ม");

    std::cout << "อ็อบเจกต์ที่สร้างในหน่วยความจำที่จองเอง:\n";
    for (int i = 0; i < 3; ++i) {
        std::cout << "  [" << i << "] " << buffer[i] << "\n";
    }

    // 3) destroy: เรียก destructor ของแต่ละอ็อบเจกต์ (แต่ยังไม่คืนหน่วยความจำ)
    for (int i = 0; i < 3; ++i) {
        Traits::destroy(alloc, &buffer[i]);
    }

    // 4) deallocate: คืนหน่วยความจำดิบกลับให้ระบบ ต้องระบุจำนวนเดิมที่ allocate ไว้
    alloc.deallocate(buffer, 3);

    std::cout << "\nจองและคืนหน่วยความจำด้วยตัวเองครบทั้ง 4 ขั้นตอนแล้ว (allocate/construct/destroy/deallocate)\n";

    return 0;
}
```

ผลลัพธ์:

```
อ็อบเจกต์ที่สร้างในหน่วยความจำที่จองเอง:
  [0] แอปเปิ้ล
  [1] กล้วย
  [2] ส้ม

จองและคืนหน่วยความจำด้วยตัวเองครบทั้ง 4 ขั้นตอนแล้ว (allocate/construct/destroy/deallocate)
```

สังเกตว่าเราเรียก `Traits::construct`/`Traits::destroy` ผ่าน `std::allocator_traits`
แทนที่จะเรียก `alloc.construct(...)` ตรงๆ — นี่คือแนวทางมาตรฐานตั้งแต่ C++11 เป็นต้นมา
เพราะ `std::allocator_traits<Alloc>` จะ "เติมเต็ม" เมธอดที่ allocator ของเราไม่ได้เขียนไว้
ให้อัตโนมัติ (เช่นถ้า allocator ของเราไม่มี `construct` เขียนเอง `allocator_traits` จะ
สร้าง fallback ที่เรียก placement `new` ให้แทน) ทำให้เราเขียน custom allocator ได้สั้นลง
มาก — ไม่ต้องเขียนทุกเมธอดให้ครบเหมือนสมัยก่อน C++11

> **หมายเหตุ**: ในทางปฏิบัติเราแทบไม่เคยเรียก `allocate`/`construct`/`destroy`/`deallocate`
> ด้วยตัวเองแบบนี้ตรงๆ (container ของ STL ทำหน้าที่นี้ให้เราอยู่แล้ว) ตัวอย่างข้างบนนี้
> มีไว้เพื่อเปิดฝาดู "เครื่องยนต์" ข้างในเท่านั้น เพื่อให้เข้าใจว่าตอนที่เราเขียน
> `v.push_back(x)` เบื้องหลังมันเกิดอะไรขึ้นบ้าง

---

## 66.3 Allocator Interface ขั้นต่ำตามมาตรฐาน C++17 (Step 523)

ถ้าจะเขียน allocator ของตัวเอง ต้องรู้ว่า container คาดหวัง interface อะไรบ้าง ข่าวดีคือ
ตั้งแต่ C++11 เป็นต้นมา (และยังใช้ได้เต็มรูปแบบใน C++17) `std::allocator_traits` ช่วยลด
สิ่งที่ต้องเขียนเองลงเหลือเพียง **4 อย่างเท่านั้น**:

| ต้องมี | รายละเอียด |
|---|---|
| `using value_type = T;` | type alias บอกว่า allocator นี้จัดสรรหน่วยความจำสำหรับชนิดข้อมูลอะไร |
| `T* allocate(std::size_t n)` | คืน pointer ไปยังหน่วยความจำดิบที่จองไว้พอสำหรับ `n` อ็อบเจกต์ |
| `void deallocate(T* ptr, std::size_t n)` | คืนหน่วยความจำที่ `allocate` เคยให้ไปกลับคืน |
| converting constructor ข้าม type | `template <typename U> Alloc(const Alloc<U>&)` สำหรับ rebind (ดู 66.6) |

นอกจากนี้ตามข้อกำหนดของ **Allocator named requirement** ยังต้องมี `operator==` และ
`operator!=` ระหว่าง allocator สองตัว (เพื่อบอกว่าหน่วยความจำที่จัดสรรจาก allocator ตัวหนึ่ง
สามารถคืนผ่านอีกตัวหนึ่งได้อย่างปลอดภัยหรือไม่) — สิ่งอื่นๆ ทั้งหมด (`construct`, `destroy`,
`rebind`, `pointer`, `size_type` ฯลฯ) `std::allocator_traits` จะสร้างค่า default ที่สมเหตุ
สมผลให้อัตโนมัติถ้าเราไม่ได้เขียนเอง

นี่คือความแตกต่างสำคัญเทียบกับ C++03 ที่การเขียน custom allocator ต้องเขียนสมาชิกครบ
เกือบ 20 ตัว (`pointer`, `const_pointer`, `reference`, `size_type`, `difference_type`,
`rebind`, `address`, `max_size` ฯลฯ) — เป็นเหตุผลหนึ่งที่ทำให้ custom allocator มีชื่อเสีย
ว่า "เขียนยากมาก" ในอดีต แต่ตั้งแต่ C++11 เป็นต้นมา ความซับซ้อนนั้นลดลงไปมากแล้ว

---

## 66.4 ทำไมบางครั้งต้องเขียน Custom Allocator (Step 524)

`std::allocator` เริ่มต้นที่ห่อ `new`/`delete` ไว้เพียงพอสำหรับงานส่วนใหญ่ในชีวิตจริง
มากกว่า 95% แต่มีบางสถานการณ์ที่วิศวกรเลือกเขียน allocator ของตัวเองด้วยเหตุผลเฉพาะ:

1. **Performance-critical code** — ถ้าโปรแกรมสร้าง/ทำลาย element จำนวนมากซ้ำๆ อย่าง
   รวดเร็ว (เช่น game engine ที่สร้าง particle หลายพันตัวทุกเฟรม, high-frequency trading
   system) การเรียก `malloc`/`free` ของระบบปฏิบัติการซ้ำๆ มี overhead ที่สะสมแล้วมีนัยสำคัญ
   (การล็อก mutex ภายใน allocator ของระบบ, การค้นหา free block ที่เหมาะสม ฯลฯ)
   Pool allocator ที่จองพื้นที่ก้อนใหญ่ไว้ล่วงหน้าแล้วแจกจ่ายเร็วๆ ช่วยลด overhead นี้ได้มาก

2. **Memory Pool / Arena Allocation** — งานที่มีรูปแบบการใช้หน่วยความจำแบบ "สร้างพร้อมกัน
   ทำลายพร้อมกัน" ชัดเจน (เช่นประมวลผล 1 HTTP request แล้วทิ้งข้อมูลทั้งหมดทีเดียวตอนจบ)
   การใช้ arena allocator ที่คืนหน่วยความจำทั้งก้อนทีเดียวเร็วกว่าการเรียก `free` ทีละ
   อ็อบเจกต์มาก

3. **Embedded System ที่ไม่มี heap ปกติ** — ระบบฝังตัวบางระบบ (เช่นไมโครคอนโทรลเลอร์ที่มี
   RAM จำกัดมาก) อาจไม่มี heap แบบที่ระบบปฏิบัติการทั่วไปมี หรือห้ามใช้ dynamic allocation
   เลยตามข้อกำหนดด้านความปลอดภัย (เช่นมาตรฐาน MISRA C++ ในอุตสาหกรรมยานยนต์/การบิน)
   custom allocator ที่ทำงานบน static memory buffer ที่กำหนดขนาดตายตัวไว้ล่วงหน้า
   ทำให้ยังใช้ STL container ได้โดยไม่ละเมิดข้อกำหนดเหล่านั้น

4. **Debugging และ Memory Tracking** — allocator ที่ log ทุกครั้งที่มีการ allocate/
   deallocate ช่วยตรวจจับ memory leak หรือวิเคราะห์ pattern การใช้หน่วยความจำของโปรแกรม
   ได้ (เราจะเห็นตัวอย่างนี้ใน 66.7)

### ตารางสรุป: ควรเขียน custom allocator เมื่อไหร่

| สถานการณ์ | ควรเขียนเองหรือไม่ |
|---|---|
| โปรแกรมทั่วไป, CRUD application, web backend | **ไม่ควร** — `std::allocator` เพียงพอ |
| Game engine ที่ profile แล้วเจอคอขวดที่ allocator จริงๆ | ควรพิจารณา |
| Embedded/real-time system ที่ห้าม dynamic allocation | จำเป็น |
| ต้องการ debug memory leak / วิเคราะห์ allocation pattern | มีประโยชน์ (logging allocator) |
| "คิดว่าน่าจะเร็วขึ้น" โดยไม่ได้ profile ก่อน | **ไม่ควร** — วัดผลจริงก่อนเสมอ |

---

## 66.5 เขียน Pool Allocator แบบง่ายๆ (Step 525)

มาลงมือเขียน custom allocator กันจริงๆ เราจะเขียน **bump allocator** (เรียกอีกชื่อว่า
arena allocator) ซึ่งเป็นรูปแบบ pool allocator ที่ง่ายที่สุดที่เป็นไปได้: จองหน่วยความจำ
ก้อนใหญ่ไว้ล่วงหน้าครั้งเดียว แล้วแจกจ่ายพื้นที่ย่อยๆ ออกไปโดยแค่ "เดินหน้า" ตัวชี้ตำแหน่ง
(offset) ไปเรื่อยๆ โดยไม่มีการคืนพื้นที่ทีละก้อนกลับมาใช้ซ้ำเลย (deallocate จริงๆ)
เหมาะกับกรณีที่อายุของอ็อบเจกต์ทั้งหมดสั้นและถูกทำลายพร้อมกันทีเดียว

เราจะแยกการออกแบบเป็นสองส่วน:

1. **`MemoryPool`** — คลาสที่เป็นเจ้าของ buffer จริงและทำหน้าที่แจกจ่าย/ติดตามพื้นที่
2. **`PoolAllocator<T>`** — ตัว "แปลงร่าง" (adapter) บางๆ ที่ทำให้ `MemoryPool` ใช้งานร่วมกับ
   STL container ได้ตาม Allocator interface ใน 66.3

การแยกแบบนี้สำคัญมาก เพราะ container อาจต้องสร้าง `PoolAllocator<U>` ที่เป็นคนละ type
กัน (ผ่าน rebind ใน 66.6) แต่ทั้งหมดต้องยังคงใช้ `MemoryPool` **ก้อนเดียวกัน** ร่วมกันอยู่ —
ถ้าให้ `PoolAllocator<T>` เป็นเจ้าของ buffer เองตรงๆ (ไม่แยกออกมาเป็น `MemoryPool`)
การ rebind แต่ละครั้งจะได้ buffer คนละก้อนกัน ซึ่งผิดจุดประสงค์ทั้งหมด

```cpp
#include <cstddef>
#include <iostream>
#include <new>
#include <vector>

// MemoryPool: ก้อนหน่วยความจำขนาดคงที่ที่จองไว้ล่วงหน้าครั้งเดียว (Arena)
// แจกจ่ายพื้นที่ย่อยๆ ให้ผู้ขอแบบ "bump pointer" (เดินหน้าเรื่อยๆ ไม่มีการคืนพื้นที่ทีละก้อน)
// นี่คือรูปแบบ pool allocator ที่ง่ายที่สุดที่เขียนได้ — เหมาะกับกรณีที่รู้ว่าอายุของอ็อบเจกต์
// ทั้งหมดจะสั้นและปล่อยพร้อมกันทีเดียว (เช่น ประมวลผล 1 request แล้วทิ้งทั้งพูล)
class MemoryPool {
public:
    explicit MemoryPool(std::size_t size_bytes)
        : buffer_(new unsigned char[size_bytes]), size_(size_bytes) {}

    ~MemoryPool() { delete[] buffer_; }

    MemoryPool(const MemoryPool&) = delete;
    MemoryPool& operator=(const MemoryPool&) = delete;

    void* allocate(std::size_t bytes, std::size_t alignment) {
        std::size_t aligned_offset = align_up(offset_, alignment);
        if (aligned_offset + bytes > size_) {
            throw std::bad_alloc();  // พูลเต็ม — งานจริงอาจสลับไปขอจาก heap แทน แต่ตัวอย่างนี้ของ่ายๆ
        }
        void* ptr = buffer_ + aligned_offset;
        offset_ = aligned_offset + bytes;
        ++allocation_count_;
        return ptr;
    }

    // pool แบบ bump allocator นี้ไม่คืนพื้นที่ทีละก้อนให้นำกลับมาใช้ซ้ำ
    // จะคืนพื้นที่ทั้งหมดพร้อมกันตอน MemoryPool ถูกทำลายเท่านั้น
    void deallocate(void*, std::size_t) noexcept {}

    std::size_t allocation_count() const noexcept { return allocation_count_; }
    std::size_t bytes_used() const noexcept { return offset_; }

private:
    static std::size_t align_up(std::size_t value, std::size_t alignment) {
        return (value + alignment - 1) / alignment * alignment;
    }

    unsigned char* buffer_;
    std::size_t size_;
    std::size_t offset_ = 0;
    std::size_t allocation_count_ = 0;
};

// PoolAllocator<T>: ตัว "แปลงร่าง" ให้ MemoryPool ใช้งานร่วมกับ STL container ได้
// ตาม Allocator concept ของ C++ ต้องมีอย่างน้อย: value_type, allocate(), deallocate(),
// converting constructor ข้าม type (สำหรับ rebind), และ operator==/operator!=
template <typename T>
class PoolAllocator {
public:
    using value_type = T;

    explicit PoolAllocator(MemoryPool& pool) noexcept : pool_(&pool) {}

    // converting constructor: จำเป็นเพราะ container ภายในอาจต้อง rebind allocator
    // ไปเป็น type อื่น (เช่น std::list<T> ต้องการ allocator ของ node ไม่ใช่ของ T ตรงๆ)
    template <typename U>
    PoolAllocator(const PoolAllocator<U>& other) noexcept : pool_(other.pool_) {}

    T* allocate(std::size_t n) {
        void* raw = pool_->allocate(n * sizeof(T), alignof(T));
        return static_cast<T*>(raw);
    }

    void deallocate(T* ptr, std::size_t n) noexcept {
        pool_->deallocate(ptr, n * sizeof(T));
    }

    MemoryPool* pool_;  // เข้าถึงได้จาก converting constructor ของ PoolAllocator<U> อื่นๆ ผ่าน friend

    template <typename U>
    friend class PoolAllocator;
};

template <typename T, typename U>
bool operator==(const PoolAllocator<T>& a, const PoolAllocator<U>& b) noexcept {
    // allocator สองตัวถือว่า "เท่ากัน" ถ้าใช้ MemoryPool ก้อนเดียวกัน (แปลว่าคืนหน่วยความจำข้ามกันได้)
    return a.pool_ == b.pool_;
}

template <typename T, typename U>
bool operator!=(const PoolAllocator<T>& a, const PoolAllocator<U>& b) noexcept {
    return !(a == b);
}

int main() {
    MemoryPool pool(1024);              // จองพูลขนาด 1024 ไบต์ไว้ล่วงหน้าครั้งเดียว
    PoolAllocator<int> alloc(pool);     // allocator ที่ผูกกับพูลนี้

    std::vector<int, PoolAllocator<int>> numbers(alloc);
    for (int i = 1; i <= 10; ++i) {
        numbers.push_back(i * i);
    }

    std::cout << "เนื้อหาใน vector: ";
    for (int n : numbers) {
        std::cout << n << " ";
    }
    std::cout << "\n";

    std::cout << "จำนวนครั้งที่ MemoryPool ถูกเรียก allocate: " << pool.allocation_count() << "\n";
    std::cout << "จำนวนไบต์ที่ใช้ไปในพูล: " << pool.bytes_used() << " / 1024 ไบต์\n";

    return 0;
}
```

ผลลัพธ์:

```
เนื้อหาใน vector: 1 4 9 16 25 36 49 64 81 100
จำนวนครั้งที่ MemoryPool ถูกเรียก allocate: 5
จำนวนไบต์ที่ใช้ไปในพูล: 124 / 1024 ไบต์
```

**สังเกตตัวเลข 5 ครั้งของการ allocate**: `std::vector` ขยายความจุแบบ exponential growth
(เพิ่มเป็นสองเท่าทุกครั้งที่เต็ม) เมื่อ push 10 ตัวเลขเข้าไปทีละตัว capacity จะไล่เป็น
1 → 2 → 4 → 8 → 16 รวมเป็น 5 ครั้งที่ต้องขอหน่วยความจำก้อนใหม่ (แต่ละครั้งใหญ่กว่าเดิม
เป็นสองเท่า) ผลรวมไบต์ที่ใช้คือ `4+8+16+32+64 = 124` ไบต์ (เพราะ `int` มีขนาด 4 ไบต์ในเครื่อง
ส่วนใหญ่) ตรงกับตัวเลขที่พิมพ์ออกมาพอดี — และเพราะ `PoolAllocator` ของเราเป็น bump
allocator ที่ไม่คืนพื้นที่เก่ากลับมาใช้ซ้ำ พื้นที่ทั้ง 4+8+16+32 ไบต์จากการ reallocate
ครั้งก่อนๆ จึงถูก "ทิ้งค้าง" ไว้ในพูลโดยไม่ได้ใช้งานอีก (นี่คือข้อจำกัดที่ยอมรับได้สำหรับ
bump allocator เพราะจุดประสงค์ของมันคือความเร็วในการจัดสรร ไม่ใช่การใช้พื้นที่อย่างคุ้มค่า
ที่สุด)

---

## 66.6 Allocator Rebind: ใช้กับ Container แบบ Node-Based (Step 526)

`std::vector<T, Alloc>` ใช้ `Alloc` ในการจัดสรรพื้นที่สำหรับ `T` โดยตรง เพราะข้อมูลของ
`vector` เก็บเป็น array ต่อเนื่องของ `T` ล้วนๆ แต่ container แบบ **node-based** อย่าง
`std::list<T, Alloc>` หรือ `std::map<K, V, Cmp, Alloc>` ไม่ได้เก็บ `T` ตรงๆ แต่เก็บเป็น
**node** ที่มีทั้งข้อมูล `T` และ pointer เชื่อมโยงไป node ข้างเคียง (เช่น `prev`/`next`
สำหรับ doubly linked list) ดังนั้น container เหล่านี้ต้องการ allocator ที่จัดสรรพื้นที่
สำหรับ **ชนิด node** ไม่ใช่ `Alloc<T>` ตรงๆ

นี่คือที่มาของ **rebind**: กลไกที่ "แปลง" `Allocator<T>` ให้กลายเป็น `Allocator<NodeType>`
โดยอัตโนมัติ ในสมัย C++03 ต้องเขียน `template <typename U> struct rebind { using other =
Allocator<U>; };` ไว้ในคลาส allocator เองตรงๆ แต่ตั้งแต่ C++11 เป็นต้นมา ถ้า allocator
ของเราเป็น class template ที่รับ type แรกเป็นชนิดข้อมูล (แบบที่เราเขียนไว้)
`std::allocator_traits` จะ **สังเคราะห์ (synthesize) rebind ให้อัตโนมัติ** โดยอาศัย
converting constructor ที่เราเขียนไว้แล้วใน 66.5 — ไม่ต้องเขียน `rebind` เองอีกต่อไป

```cpp
#include <cstddef>
#include <iostream>
#include <list>
#include <memory>
#include <new>

// (MemoryPool และ PoolAllocator เหมือนกับใน 66.5 ทุกประการ)
class MemoryPool {
public:
    explicit MemoryPool(std::size_t size_bytes)
        : buffer_(new unsigned char[size_bytes]), size_(size_bytes) {}
    ~MemoryPool() { delete[] buffer_; }
    MemoryPool(const MemoryPool&) = delete;
    MemoryPool& operator=(const MemoryPool&) = delete;

    void* allocate(std::size_t bytes, std::size_t alignment) {
        std::size_t aligned_offset = align_up(offset_, alignment);
        if (aligned_offset + bytes > size_) {
            throw std::bad_alloc();
        }
        void* ptr = buffer_ + aligned_offset;
        offset_ = aligned_offset + bytes;
        ++allocation_count_;
        return ptr;
    }
    void deallocate(void*, std::size_t) noexcept {}
    std::size_t allocation_count() const noexcept { return allocation_count_; }

private:
    static std::size_t align_up(std::size_t v, std::size_t a) { return (v + a - 1) / a * a; }
    unsigned char* buffer_;
    std::size_t size_;
    std::size_t offset_ = 0;
    std::size_t allocation_count_ = 0;
};

template <typename T>
class PoolAllocator {
public:
    using value_type = T;
    explicit PoolAllocator(MemoryPool& pool) noexcept : pool_(&pool) {}
    template <typename U>
    PoolAllocator(const PoolAllocator<U>& other) noexcept : pool_(other.pool_) {}

    T* allocate(std::size_t n) { return static_cast<T*>(pool_->allocate(n * sizeof(T), alignof(T))); }
    void deallocate(T* ptr, std::size_t n) noexcept { pool_->deallocate(ptr, n * sizeof(T)); }

    MemoryPool* pool_;
    template <typename U>
    friend class PoolAllocator;
};

template <typename T, typename U>
bool operator==(const PoolAllocator<T>& a, const PoolAllocator<U>& b) noexcept { return a.pool_ == b.pool_; }
template <typename T, typename U>
bool operator!=(const PoolAllocator<T>& a, const PoolAllocator<U>& b) noexcept { return !(a == b); }

int main() {
    // std::list<int> ไม่ได้เก็บ int ตรงๆ แต่เก็บเป็น "node" ที่มีทั้งค่า int และ pointer ไป node ข้างเคียง
    // ดังนั้น list ต้องขอ allocator สำหรับ "ชนิด node" ไม่ใช่ PoolAllocator<int> ตรงๆ
    // มันจะใช้ std::allocator_traits<PoolAllocator<int>>::rebind_alloc<NodeType> เพื่อ "แปลง" ชนิด
    // ให้อัตโนมัติ โดยอาศัย converting constructor ที่เราเขียนไว้ — เราไม่ต้องเขียน rebind member เอง

    using ReboundType = std::allocator_traits<PoolAllocator<int>>::rebind_alloc<double>;
    static_assert(std::is_same_v<ReboundType, PoolAllocator<double>>,
                  "allocator_traits ควร rebind PoolAllocator<int> เป็น PoolAllocator<double> ได้เอง");

    MemoryPool pool(4096);
    PoolAllocator<int> alloc(pool);

    std::list<int, PoolAllocator<int>> numbers(alloc);
    for (int i = 1; i <= 5; ++i) {
        numbers.push_back(i * 10);
    }

    std::cout << "เนื้อหาใน list: ";
    for (int n : numbers) {
        std::cout << n << " ";
    }
    std::cout << "\n";
    std::cout << "จำนวนครั้งที่ MemoryPool ถูก allocate (แต่ละ node คือ 1 การขอ): "
              << pool.allocation_count() << "\n";

    return 0;
}
```

ผลลัพธ์:

```
เนื้อหาใน list: 10 20 30 40 50
จำนวนครั้งที่ MemoryPool ถูก allocate (แต่ละ node คือ 1 การขอ): 5
```

สังเกตว่าเราไม่ได้เขียน `PoolAllocator<int>` ไปใช้กับ `std::list<int, PoolAllocator<int>>`
ตรงๆ แบบผิวเผิน — เบื้องหลัง `std::list` จะ rebind `PoolAllocator<int>` ให้กลายเป็น
`PoolAllocator<ListNode<int>>` (ชื่อ node type จริงเป็น implementation detail ของแต่ละ
compiler) โดยอัตโนมัติผ่าน `std::allocator_traits::rebind_alloc` แล้วใช้ allocator ที่ถูก
rebind นั้นจัดสรรพื้นที่สำหรับ node แต่ละตัว และเพราะ `MemoryPool` เดียวกันถูกแชร์ผ่าน
pointer (`pool_`) ในทุก instance ของ `PoolAllocator<U>` ที่ rebind ออกมา การ allocate node
ทุกตัวจึงยังคงมาจากพูลก้อนเดียวกันเสมอ — นี่คือเหตุผลที่การแยก `MemoryPool` ออกจาก
`PoolAllocator` (แทนที่จะให้ `PoolAllocator` เป็นเจ้าของ buffer เอง) มีความสำคัญมาก

จำนวนครั้งที่ `allocate` = 5 ตรงกับจำนวน node ที่สร้าง (5 ตัวเลขที่ push เข้าไป) เพราะ
`std::list` จองพื้นที่ทีละ 1 node ต่อการ `push_back` หนึ่งครั้ง ต่างจาก `std::vector` ใน
66.5 ที่ขยายเป็นก้อนใหญ่ล่วงหน้าตาม growth factor

---

## 66.7 เปรียบเทียบพฤติกรรมของ Allocator ด้วย Logging Allocator (Step 527)

ก่อนตัดสินใจเขียน custom allocator เพื่อ "เพิ่มประสิทธิภาพ" ควรวัดผลจริงก่อนเสมอ — และ
วิธีที่ตรงไปตรงมาที่สุดในการดูว่า container เรียก allocate/deallocate บ่อยแค่ไหนคือเขียน
**Logging Allocator**: allocator ที่ห่อ `std::allocator` เดิมไว้ แต่แอบพิมพ์ log ทุกครั้ง
ที่ถูกเรียกใช้งาน

```cpp
#include <iostream>
#include <memory>
#include <vector>

// LoggingAllocator<T>: ห่อ std::allocator<T> เดิมไว้ แล้วแค่ "แอบดัก" ทุกครั้งที่ container
// เรียก allocate/deallocate เพื่อให้เห็นภาพว่า std::vector ขยายหน่วยความจำ (reallocate) เมื่อไหร่
template <typename T>
class LoggingAllocator {
public:
    using value_type = T;

    LoggingAllocator() noexcept = default;

    template <typename U>
    LoggingAllocator(const LoggingAllocator<U>&) noexcept {}

    T* allocate(std::size_t n) {
        std::cout << "  [allocate]   ขอพื้นที่สำหรับ " << n << " element(s) ("
                  << (n * sizeof(T)) << " ไบต์)\n";
        return std::allocator<T>{}.allocate(n);
    }

    void deallocate(T* ptr, std::size_t n) noexcept {
        std::cout << "  [deallocate] คืนพื้นที่ของ " << n << " element(s) ("
                  << (n * sizeof(T)) << " ไบต์)\n";
        std::allocator<T>{}.deallocate(ptr, n);
    }
};

template <typename T, typename U>
bool operator==(const LoggingAllocator<T>&, const LoggingAllocator<U>&) noexcept {
    return true;  // stateless allocator: ทุก instance ใช้แทนกันได้เสมอ
}

template <typename T, typename U>
bool operator!=(const LoggingAllocator<T>& a, const LoggingAllocator<U>& b) noexcept {
    return !(a == b);
}

int main() {
    std::cout << "push_back ทีละตัว 8 ครั้ง สังเกตรูปแบบการ allocate/deallocate ของ std::vector:\n\n";

    std::vector<int, LoggingAllocator<int>> v;
    for (int i = 1; i <= 8; ++i) {
        std::cout << "push_back(" << i << ")  capacity ก่อนหน้า = " << v.capacity() << "\n";
        v.push_back(i);
    }

    std::cout << "\ncapacity สุดท้าย: " << v.capacity() << ", size: " << v.size() << "\n";
    std::cout << "(เมื่อ vector ออกจาก scope จะเห็น deallocate ครั้งสุดท้ายตามมาอีกหนึ่งครั้ง)\n";

    return 0;
}
```

ผลลัพธ์:

```
push_back ทีละตัว 8 ครั้ง สังเกตรูปแบบการ allocate/deallocate ของ std::vector:

push_back(1)  capacity ก่อนหน้า = 0
  [allocate]   ขอพื้นที่สำหรับ 1 element(s) (4 ไบต์)
push_back(2)  capacity ก่อนหน้า = 1
  [allocate]   ขอพื้นที่สำหรับ 2 element(s) (8 ไบต์)
  [deallocate] คืนพื้นที่ของ 1 element(s) (4 ไบต์)
push_back(3)  capacity ก่อนหน้า = 2
  [allocate]   ขอพื้นที่สำหรับ 4 element(s) (16 ไบต์)
  [deallocate] คืนพื้นที่ของ 2 element(s) (8 ไบต์)
push_back(4)  capacity ก่อนหน้า = 4
push_back(5)  capacity ก่อนหน้า = 4
  [allocate]   ขอพื้นที่สำหรับ 8 element(s) (32 ไบต์)
  [deallocate] คืนพื้นที่ของ 4 element(s) (16 ไบต์)
push_back(6)  capacity ก่อนหน้า = 8
push_back(7)  capacity ก่อนหน้า = 8
push_back(8)  capacity ก่อนหน้า = 8

capacity สุดท้าย: 8, size: 8
(เมื่อ vector ออกจาก scope จะเห็น deallocate ครั้งสุดท้ายตามมาอีกหนึ่งครั้ง)
  [deallocate] คืนพื้นที่ของ 8 element(s) (32 ไบต์)
```

จะเห็นแพทเทิร์นชัดเจนมาก: `std::vector` ของ libstdc++ ขยาย capacity เป็น **สองเท่า**
ทุกครั้งที่เต็ม (1 → 2 → 4 → 8) และทุกครั้งที่ reallocate จะมีทั้ง `allocate` (พื้นที่ใหม่
ที่ใหญ่กว่า) ตามด้วย `deallocate` (พื้นที่เก่าที่ไม่ใช้แล้ว หลัง copy/move ข้อมูลเก่าไปที่
ใหม่เสร็จ) เทคนิคแบบ `LoggingAllocator` นี้มีประโยชน์มากในการ debug ปัญหาประสิทธิภาพจริง
เช่น ถ้าพบว่าโปรแกรมมีการ reallocate บ่อยผิดปกติ อาจแก้ปัญหาได้ง่ายๆ ด้วยการเรียก
`v.reserve(expected_size)` ล่วงหน้า แทนที่จะต้องเขียน custom allocator ที่ซับซ้อนเลย

---

## 66.8 ข้อควรระวังและความซับซ้อนของ Custom Allocator (Step 528)

หลังจากเห็นตัวอย่างที่ใช้งานได้จริงแล้ว มาถึงส่วนที่สำคัญที่สุดของบทนี้: **ทำไมงานจริง
ส่วนใหญ่ไม่ควรเขียน custom allocator เอง** และมีความซับซ้อนอะไรซ่อนอยู่บ้างที่ตัวอย่าง
ข้างบนยังไม่ได้พูดถึง

### 1. Stateful Allocator ทำให้พฤติกรรมของ container ซับซ้อนขึ้นมาก

`PoolAllocator` ของเราเป็น **stateful allocator** (มีสถานะภายในคือ pointer ไปยัง
`MemoryPool`) ต่างจาก `std::allocator` เริ่มต้นที่เป็น **stateless** (ไม่มีสถานะใดๆ
ทุก instance เหมือนกันหมด) ความแตกต่างนี้ส่งผลกระทบมากกว่าที่คิด:

- การ **copy container** ที่ใช้ stateful allocator ต้องตัดสินใจว่า container ใหม่จะ
  ใช้ allocator (และ pool) เดียวกับต้นฉบับ หรือสร้าง allocator ใหม่ — พฤติกรรมนี้ควบคุม
  ผ่าน `select_on_container_copy_construction` ซึ่งเป็นรายละเอียดขั้นสูงที่ต้อง implement
  เพิ่มถ้าต้องการควบคุมพฤติกรรมนี้อย่างถูกต้อง
- การ **move/swap container** ที่ใช้ allocator ต่างกัน (คนละ `MemoryPool`) มีกฎเกณฑ์
  ซับซ้อนเกี่ยวกับว่าจะ "ย้าย" ข้อมูลจริง (ทีละอ็อบเจกต์ เพราะข้าม allocator ไม่ได้)
  หรือจะ "สลับ pointer" (เร็วกว่ามาก) ซึ่งควบคุมผ่าน
  `propagate_on_container_move_assignment` และ `propagate_on_container_swap`
  ถ้าไม่ระบุ traits เหล่านี้ให้ถูกต้อง อาจได้พฤติกรรมที่ไม่คาดคิดหรือ Undefined Behavior

### 2. ปัญหาเรื่องอายุการใช้งาน (Lifetime) ของ MemoryPool

ตัวอย่างใน 66.5 มีข้อจำกัดที่ซ่อนอยู่: `MemoryPool` ต้องมีชีวิตอยู่ **นานกว่า** container
ทุกตัวที่ใช้ allocator ที่ผูกกับมันเสมอ ถ้า `MemoryPool` ถูกทำลายไปแล้วแต่ `std::vector`
ที่ใช้ allocator ของมันยังไม่ถูกทำลาย การเรียก `push_back` หรือแม้แต่ destructor ของ
`vector` (ที่ต้องเรียก `deallocate`) จะกลายเป็น Undefined Behavior ทันที — เป็นปัญหา
เรื่อง dangling pointer แบบเดียวกับที่เราเจอกับ `string_view` ใน Part 65 เพียงแต่คราวนี้
เกิดกับ allocator แทน

### 3. Exception Safety ระหว่างการจัดสรร

ถ้า `allocate` ของ custom allocator throw exception (เช่นพูลเต็มแล้วใน 66.5) container
ต้องมั่นใจว่าจะไม่เกิด memory leak หรือสถานะที่เสียหาย (เช่น element ที่สร้างไปแล้วบางส่วน
ต้องถูก destroy ให้ครบก่อนที่ exception จะลอยขึ้นไปถึงผู้เรียก) STL container มาตรฐาน
จัดการเรื่องนี้ให้อัตโนมัติถ้า allocator ของเรา throw exception ตามข้อกำหนดที่ถูกต้อง
(เช่น `allocate` throw `std::bad_alloc` เมื่อจัดสรรไม่สำเร็จ) แต่ถ้าเขียน allocator เอง
โดยไม่เข้าใจข้อกำหนดเรื่อง exception safety ให้ครบถ้วน อาจทำให้เกิดพฤติกรรมที่ผิดพลาด
ในสถานการณ์ที่ไม่ค่อยเกิดขึ้น (edge case) และตรวจจับได้ยากมากในการทดสอบทั่วไป

### 4. ผลตอบแทนที่ได้อาจไม่คุ้มกับความซับซ้อนที่เพิ่มขึ้น

นี่คือประเด็นที่สำคัญที่สุด: การเขียน custom allocator เพิ่ม **ความซับซ้อนของโค้ด** และ
**พื้นผิวสำหรับบั๊ก** อย่างมีนัยสำคัญ แลกกับ performance ที่ในหลายกรณีสามารถได้มาง่ายกว่า
มากด้วยวิธีอื่น เช่น:

- เรียก `v.reserve(n)` ล่วงหน้าถ้ารู้ขนาดโดยประมาณ (ลด reallocation ได้มากโดยไม่ต้องแตะ
  allocator เลย)
- ใช้ `std::pmr::polymorphic_allocator` (C++17, อยู่ใน `<memory_resource>`) ซึ่งเป็น
  กลไก memory pool ที่มาตรฐานเตรียมไว้ให้แล้ว ปลอดภัยกว่าและทดสอบมาอย่างดีกว่าโค้ดที่
  เขียนเอง (จะกล่าวถึงในรายละเอียดที่ Part หลังๆ ของหลักสูตรที่เกี่ยวกับ Performance)
- Profile โปรแกรมจริงก่อนเสมอ (ด้วยเครื่องมืออย่าง `perf`, Valgrind's Massif ซึ่งเรียนไป
  แล้วใน Part 38) เพื่อยืนยันว่า allocator เป็นคอขวดจริงๆ ก่อนลงทุนเขียน custom allocator

### ตารางสรุปข้อควรระวัง

| ประเด็น | ผลกระทบ |
|---|---|
| Stateful allocator | ต้องคิดเรื่อง copy/move/swap ระหว่าง allocator ต่างสถานะให้ถูกต้อง |
| Lifetime ของทรัพยากรที่ allocator ผูกอยู่ (เช่น `MemoryPool`) | ต้องมีอายุยืนกว่า container ที่ใช้มันเสมอ ไม่งั้นเกิด dangling |
| Exception safety | ต้องมั่นใจว่า `allocate` ที่ throw ไม่ทำให้เกิด memory leak หรือสถานะเสียหาย |
| ความซับซ้อนของโค้ด | เพิ่มพื้นผิวสำหรับบั๊กที่ตรวจจับยาก โดยเฉพาะ edge case ที่ไม่ค่อยเกิด |
| ทางเลือกอื่นที่ปลอดภัยกว่า | `reserve()`, `std::pmr::polymorphic_allocator`, profile ก่อนตัดสินใจเสมอ |

> **สรุปสั้นๆ ที่ควรจำ**: เข้าใจกลไกของ allocator ให้ลึกซึ้ง (อย่างที่เราทำในบทนี้) เพราะ
> มันช่วยให้เข้าใจว่า STL container ทำงานอย่างไรเบื้องหลัง และเป็นความรู้สำคัญเวลาต้อง
> อ่านหรือ debug โค้ดของคนอื่นที่ใช้ custom allocator แต่ **ในงานจริงส่วนใหญ่ไม่จำเป็นต้อง
> เขียน custom allocator เอง** — ให้ใช้ `std::allocator` เริ่มต้นไปก่อน แล้ววัดผลจริงด้วย
> เครื่องมือ profiling ก่อนตัดสินใจลงทุนเขียนกลไกที่ซับซ้อนขึ้นเสมอ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ทำลาย `MemoryPool` (หรือทรัพยากรที่ allocator ผูกอยู่) ก่อน container ที่ใช้มัน** —
   เป็นสาเหตุอันดับหนึ่งของ Undefined Behavior เมื่อใช้ stateful allocator เสมอตรวจสอบว่า
   `MemoryPool` ประกาศไว้ **ก่อน** และมี scope **กว้างกว่า** container ที่ใช้มันเสมอ

2. **ลืมเขียน converting constructor ข้าม type** (`template <typename U> Alloc(const
   Alloc<U>&)`) — ถ้าไม่มีตัวนี้ container แบบ node-based เช่น `std::list`/`std::map`
   จะ compile ไม่ผ่านทันที เพราะ `std::allocator_traits` ไม่สามารถ rebind allocator
   ของเราไปเป็น type อื่นได้

3. **ลืมเขียน `operator==`/`operator!=`** — แม้ `std::allocator_traits` จะเติมเต็ม
   เมธอดหลายตัวให้อัตโนมัติ แต่ operator เปรียบเทียบความเท่ากันยังคงเป็นข้อกำหนดขั้นต่ำ
   ที่ต้องมีตาม Allocator named requirement เสมอ

4. **ใช้ pool allocator แบบ bump (ไม่คืนพื้นที่จริง) กับโปรแกรมที่รันระยะยาวและสร้าง/
   ทำลาย element ต่อเนื่องไม่หยุด** — พูลจะเต็มเร็วมากเพราะไม่มีการนำพื้นที่เก่ากลับมาใช้
   ซ้ำเลย เหมาะกับงานที่มีจุดจบชัดเจน (เช่นประมวลผล 1 request แล้วรีเซ็ตพูลทั้งหมด)
   มากกว่างานที่รันตลอดอายุโปรแกรม

5. **เข้าใจผิดว่า custom allocator จะเร็วกว่า `std::allocator` เสมอ** — `std::allocator`
   เริ่มต้นของ compiler สมัยใหม่ผ่านการ optimize มาอย่างดีมาก (เช่น libstdc++ มี fast
   path สำหรับ block ขนาดเล็กอยู่แล้ว) การเขียน custom allocator ที่ไม่ได้ออกแบบอย่าง
   รอบคอบอาจช้ากว่าของเดิมด้วยซ้ำ ต้อง benchmark เปรียบเทียบจริงเสมอ ไม่ใช่เดา

6. **ไม่ได้พิจารณาเรื่อง alignment** — บาง type (เช่น SIMD type อย่าง `__m128`) ต้องการ
   หน่วยความจำที่ align ตามค่าที่มากกว่าค่า default ของระบบ ถ้า custom allocator ของเรา
   ไม่ได้เรียก `alignof(T)` แล้วจัดสรรให้ตรงตามนั้น (ตามตัวอย่างใน `align_up` ของเรา)
   อาจทำให้เกิด crash หรือ performance ตกลงบน CPU บางรุ่นที่บังคับเรื่อง alignment
   เข้มงวด

---

## แบบฝึกหัดท้ายบท

1. อธิบายด้วยคำพูดตัวเองว่าทำไม `deallocate` ของ `PoolAllocator` ใน 66.5 ถึงไม่ได้คืน
   พื้นที่ให้นำกลับมาใช้ซ้ำจริงๆ และอะไรจะเกิดขึ้นถ้าพูลใช้พื้นที่จนเต็มระหว่างการทำงาน
2. เขียนโปรแกรมทดสอบว่าเมื่อ `MemoryPool` มีขนาดเล็กเกินไป (เช่น 32 ไบต์) การ `push_back`
   เข้า `std::vector<int, PoolAllocator<int>>` ต่อเนื่องจะ throw `std::bad_alloc`
   จริงหรือไม่ พร้อมจับ exception นั้นด้วย `try`/`catch`
3. ปรับปรุง `LoggingAllocator` จาก 66.7 ให้เก็บสถิติจำนวนครั้งที่ `allocate` ถูกเรียก
   ทั้งหมดตลอดโปรแกรม (ใช้ตัวแปร `static`) แล้วพิมพ์สรุปตอนจบโปรแกรม
4. เขียนโปรแกรมที่สร้าง `MemoryPool` ก้อนเดียว แล้วใช้ `PoolAllocator` ตัวเดียวกันนั้นกับ
   ทั้ง `std::vector<int, PoolAllocator<int>>` และ `std::vector<double,
   PoolAllocator<double>>` (ผ่าน converting constructor) พร้อมยืนยันว่าทั้งสอง container
   แชร์การนับ `allocation_count()` จาก `MemoryPool` เดียวกัน
5. เขียนโปรแกรมเปรียบเทียบเวลาที่ใช้ (ด้วย `std::chrono`) ระหว่างการสร้างและทำลาย
   `std::vector<int>` ขนาดเล็กซ้ำๆ หลายหมื่นรอบ โดยเทียบระหว่าง `std::allocator` เริ่มต้น
   กับ `PoolAllocator` ที่ `reset()` พูลใหม่ทุกรอบ
6. อธิบายว่าทำไม `MemoryPool` ในบทเรียนนี้ถูกกำหนดให้ **ห้าม copy** (`= delete` ทั้ง copy
   constructor และ copy assignment) และจะเกิดปัญหาอะไรถ้าเราลบข้อจำกัดนี้ออกแล้วปล่อยให้
   copy ได้ตามปกติ

### แนวทางเฉลยข้อ 2

```cpp
#include <cstddef>
#include <iostream>
#include <new>
#include <stdexcept>
#include <vector>

// (คัดลอก MemoryPool / PoolAllocator มาจากเนื้อหาบทเรียน)
class MemoryPool {
public:
    explicit MemoryPool(std::size_t size_bytes)
        : buffer_(new unsigned char[size_bytes]), size_(size_bytes) {}
    ~MemoryPool() { delete[] buffer_; }
    MemoryPool(const MemoryPool&) = delete;
    MemoryPool& operator=(const MemoryPool&) = delete;

    void* allocate(std::size_t bytes, std::size_t alignment) {
        std::size_t aligned_offset = align_up(offset_, alignment);
        if (aligned_offset + bytes > size_) {
            throw std::bad_alloc();
        }
        void* ptr = buffer_ + aligned_offset;
        offset_ = aligned_offset + bytes;
        return ptr;
    }
    void deallocate(void*, std::size_t) noexcept {}

private:
    static std::size_t align_up(std::size_t v, std::size_t a) { return (v + a - 1) / a * a; }
    unsigned char* buffer_;
    std::size_t size_;
    std::size_t offset_ = 0;
};

template <typename T>
class PoolAllocator {
public:
    using value_type = T;
    explicit PoolAllocator(MemoryPool& pool) noexcept : pool_(&pool) {}
    template <typename U>
    PoolAllocator(const PoolAllocator<U>& other) noexcept : pool_(other.pool_) {}

    T* allocate(std::size_t n) { return static_cast<T*>(pool_->allocate(n * sizeof(T), alignof(T))); }
    void deallocate(T* ptr, std::size_t n) noexcept { pool_->deallocate(ptr, n * sizeof(T)); }

    MemoryPool* pool_;
    template <typename U>
    friend class PoolAllocator;
};

template <typename T, typename U>
bool operator==(const PoolAllocator<T>& a, const PoolAllocator<U>& b) noexcept { return a.pool_ == b.pool_; }
template <typename T, typename U>
bool operator!=(const PoolAllocator<T>& a, const PoolAllocator<U>& b) noexcept { return !(a == b); }

int main() {
    // ตั้งใจสร้างพูลให้เล็กมาก (32 ไบต์ = เก็บ int ได้แค่ 8 ตัว) เพื่อบังคับให้มันเต็มเร็วๆ
    MemoryPool tiny_pool(32);
    PoolAllocator<int> alloc(tiny_pool);
    std::vector<int, PoolAllocator<int>> numbers(alloc);

    try {
        for (int i = 1; i <= 100; ++i) {
            numbers.push_back(i);
            std::cout << "push_back(" << i << ") สำเร็จ, capacity=" << numbers.capacity() << "\n";
        }
    } catch (const std::bad_alloc& e) {
        std::cout << "\nจับ std::bad_alloc ได้ตามคาด: พูลเต็มแล้ว (" << e.what() << ")\n";
        std::cout << "จำนวน element ที่ใส่ได้ก่อนพูลจะเต็ม: " << numbers.size() << "\n";
    }

    return 0;
}
```

ผลลัพธ์:

```
push_back(1) สำเร็จ, capacity=1
push_back(2) สำเร็จ, capacity=2
push_back(3) สำเร็จ, capacity=4
push_back(4) สำเร็จ, capacity=4

จับ std::bad_alloc ได้ตามคาด: พูลเต็มแล้ว (std::bad_alloc)
จำนวน element ที่ใส่ได้ก่อนพูลจะเต็ม: 4
```

**คำอธิบาย**: พูลขนาด 32 ไบต์ เก็บ `int` (4 ไบต์ต่อตัว) ตามหลัก exponential growth ของ
`vector`: capacity 1 ใช้ 4 ไบต์ (รวม 4), capacity 2 ใช้ 8 ไบต์ (รวม 12), capacity 4 ใช้
16 ไบต์ (รวม 28) — พอ `vector` พยายามขยายเป็น capacity 8 ซึ่งต้องการอีก 32 ไบต์ติดต่อกัน
(28 + 32 = 60 > 32 ที่มีอยู่) `MemoryPool::allocate` จึง throw `std::bad_alloc` เพราะ
พื้นที่ในพูลไม่พอ (โปรดสังเกตว่าเพราะ `PoolAllocator` เป็น bump allocator ที่ไม่คืนพื้นที่
เก่า (4+8+16=28 ไบต์จาก capacity 1,2,4 เดิม) จึงถูก "ทิ้งค้าง" ไว้ ทำให้พูลเต็มเร็วกว่าที่
คิดถ้าเทียบกับ allocator ที่นำพื้นที่เก่ากลับมาใช้ซ้ำได้จริง) — exception นี้ถูกจับได้อย่าง
ถูกต้องด้วย `try`/`catch` ตามที่ STL container คาดหวังจาก allocator ที่ดี: เมื่อ `allocate`
ล้มเหลว ต้อง throw `std::bad_alloc` เพื่อให้ `vector` จัดการ rollback สถานะให้ปลอดภัย
(ในกรณีนี้คือ `vector` ยังคงมี 4 element เดิมอยู่ครบถ้วน ไม่มีข้อมูลเสียหาย)

### แนวทางเฉลยข้อ 5

```cpp
#include <chrono>
#include <cstddef>
#include <iostream>
#include <new>
#include <vector>

class MemoryPool {
public:
    explicit MemoryPool(std::size_t size_bytes)
        : buffer_(new unsigned char[size_bytes]), size_(size_bytes) {}
    ~MemoryPool() { delete[] buffer_; }
    MemoryPool(const MemoryPool&) = delete;
    MemoryPool& operator=(const MemoryPool&) = delete;

    void* allocate(std::size_t bytes, std::size_t alignment) {
        std::size_t aligned_offset = align_up(offset_, alignment);
        if (aligned_offset + bytes > size_) {
            throw std::bad_alloc();
        }
        void* ptr = buffer_ + aligned_offset;
        offset_ = aligned_offset + bytes;
        return ptr;
    }
    void deallocate(void*, std::size_t) noexcept {}
    void reset() noexcept { offset_ = 0; }

private:
    static std::size_t align_up(std::size_t v, std::size_t a) { return (v + a - 1) / a * a; }
    unsigned char* buffer_;
    std::size_t size_;
    std::size_t offset_ = 0;
};

template <typename T>
class PoolAllocator {
public:
    using value_type = T;
    explicit PoolAllocator(MemoryPool& pool) noexcept : pool_(&pool) {}
    template <typename U>
    PoolAllocator(const PoolAllocator<U>& other) noexcept : pool_(other.pool_) {}

    T* allocate(std::size_t n) { return static_cast<T*>(pool_->allocate(n * sizeof(T), alignof(T))); }
    void deallocate(T* ptr, std::size_t n) noexcept { pool_->deallocate(ptr, n * sizeof(T)); }

    MemoryPool* pool_;
    template <typename U>
    friend class PoolAllocator;
};

template <typename T, typename U>
bool operator==(const PoolAllocator<T>& a, const PoolAllocator<U>& b) noexcept { return a.pool_ == b.pool_; }
template <typename T, typename U>
bool operator!=(const PoolAllocator<T>& a, const PoolAllocator<U>& b) noexcept { return !(a == b); }

constexpr int kRounds = 20000;
constexpr int kElementsPerVector = 16;

long long time_with_default_allocator() {
    auto start = std::chrono::steady_clock::now();
    for (int round = 0; round < kRounds; ++round) {
        std::vector<int> v;
        for (int i = 0; i < kElementsPerVector; ++i) {
            v.push_back(i);
        }
    }
    auto end = std::chrono::steady_clock::now();
    return std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
}

long long time_with_pool_allocator() {
    MemoryPool pool(1u << 16);  // 64 KB พูลที่ใช้ซ้ำ (reset ทุกรอบ) แทนการขอ/คืน heap จริงทุกครั้ง
    auto start = std::chrono::steady_clock::now();
    for (int round = 0; round < kRounds; ++round) {
        pool.reset();
        PoolAllocator<int> alloc(pool);
        std::vector<int, PoolAllocator<int>> v(alloc);
        for (int i = 0; i < kElementsPerVector; ++i) {
            v.push_back(i);
        }
    }
    auto end = std::chrono::steady_clock::now();
    return std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
}

int main() {
    long long default_time = time_with_default_allocator();
    long long pool_time = time_with_pool_allocator();

    std::cout << "std::allocator ปกติ: " << default_time << " microseconds สำหรับ "
              << kRounds << " รอบ\n";
    std::cout << "PoolAllocator (bump, reset ซ้ำ): " << pool_time << " microseconds สำหรับ "
              << kRounds << " รอบ\n";
    std::cout << "\n(ผลลัพธ์ตัวเลขจริงขึ้นกับเครื่องและ allocator ของระบบปฏิบัติการ "
              << "แต่แนวโน้มที่ควรเห็นคือ pool allocator เร็วกว่าอย่างชัดเจน "
              << "เพราะไม่ต้องเรียก malloc/free ของระบบซ้ำๆ ในทุกรอบ)\n";

    return 0;
}
```

ผลลัพธ์ตัวอย่าง (ทดสอบด้วย GCC 13, flag `-O2`, ตัวเลขจริงจะต่างกันไปตามเครื่อง):

```
std::allocator ปกติ: 2203 microseconds สำหรับ 20000 รอบ
PoolAllocator (bump, reset ซ้ำ): 641 microseconds สำหรับ 20000 รอบ

(ผลลัพธ์ตัวเลขจริงขึ้นกับเครื่องและ allocator ของระบบปฏิบัติการ แต่แนวโน้มที่ควรเห็นคือ pool allocator เร็วกว่าอย่างชัดเจน เพราะไม่ต้องเรียก malloc/free ของระบบซ้ำๆ ในทุกรอบ)
```

**คำอธิบาย**: `time_with_default_allocator` สร้าง `std::vector<int>` ใหม่ทุกรอบ (20,000
รอบ) แต่ละรอบมีการขอ/คืน heap memory ผ่าน `malloc`/`free` ของระบบปฏิบัติการหลายครั้งตาม
growth factor ในขณะที่ `time_with_pool_allocator` ใช้ `MemoryPool` ก้อนเดียวขนาด 64 KB
ตลอดการทดสอบ แล้วแค่ `reset()` (คือรีเซ็ต `offset_` กลับเป็น 0) ก่อนเริ่มรอบใหม่ทุกครั้ง
แทนที่จะขอ/คืน heap memory จริงกับระบบปฏิบัติการเลย — ผลคือเร็วกว่าอย่างเห็นได้ชัด (ในการ
ทดสอบนี้ประมาณ 3.4 เท่า) เพราะการ "รีเซ็ต offset" เป็นการดำเนินการที่เร็วกว่าการเรียก
`malloc`/`free` ของระบบมาก แต่ควรสังเกตว่านี่เป็นสถานการณ์ที่ **เหมาะกับ pool allocator
เป็นพิเศษ** (สร้าง/ทำลาย object จำนวนมากซ้ำๆ ในรูปแบบเดิม) ซึ่งไม่ใช่ทุกโปรแกรมที่มี
pattern การใช้หน่วยความจำแบบนี้ — นี่คือเหตุผลที่ต้อง benchmark สถานการณ์จริงของตัวเองเสมอ
ก่อนตัดสินใจว่าคุ้มค่าที่จะเขียน custom allocator หรือไม่

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Allocator คือกลไกที่ container ของ STL ใช้จัดสรร/คืนหน่วยความจำเบื้องหลัง
  ซึ่งถูกออกแบบให้แยกออกจาก logic ของ container อย่างชัดเจนตามหลัก Separation of Concerns
- เจาะลึกการทำงานของ `std::allocator` เริ่มต้น ผ่านการเรียก `allocate`, `construct`,
  `destroy`, `deallocate` ด้วยตัวเอง
- รู้จัก Allocator interface ขั้นต่ำของ C++17 และเข้าใจว่า `std::allocator_traits`
  ช่วยลดสิ่งที่ต้องเขียนเองลงมากเมื่อเทียบกับสมัย C++03
- เขียน pool allocator แบบง่ายของตัวเอง (`MemoryPool` + `PoolAllocator<T>`) และนำไปใช้กับ
  ทั้ง `std::vector` และ `std::list` ได้จริง พร้อมเข้าใจกลไก rebind
- ใช้ Logging Allocator สังเกตพฤติกรรมการขยายหน่วยความจำจริงของ `std::vector`
- ตระหนักถึงข้อควรระวังสำคัญของ custom allocator: stateful allocator, lifetime ของ
  ทรัพยากรที่ผูกอยู่, exception safety และเข้าใจว่าในงานจริงส่วนใหญ่ควรใช้
  `std::allocator` เริ่มต้นไปก่อน แล้ว profile ก่อนตัดสินใจเขียนเอง

ใน **Part 67** เราจะเรียนรู้เรื่อง **Smart Pointer** (`std::unique_ptr`, `std::shared_ptr`,
`std::weak_ptr`) ซึ่งเป็นเครื่องมือสำคัญที่สุดตัวหนึ่งของ Modern C++ ในการจัดการหน่วยความจำ
อัตโนมัติโดยไม่ต้องเรียก `delete` เองเลย ต่อยอดจากแนวคิดเรื่อง RAII และการจัดการทรัพยากร
ที่เราได้สัมผัสมาบ้างแล้วในบทนี้

**ต่อไป:** [Part 67 — Smart Pointer (unique/shared/weak_ptr)](./part-067-smart-pointers.md)
