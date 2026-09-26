# Part 68: RAII และ Resource Management Pattern ระดับ Production (Step 537–544)

> Module E — Templates, Generic Programming และ STL | Part 68 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 537–544
> Part ก่อนหน้า: [Part 67 — Smart Pointer (unique_ptr, shared_ptr, weak_ptr) แบบเจาะลึก](./part-067-smart-pointers.md) | Part ถัดไป: [Part 69 — ภาพรวม C++11](./part-069-cpp11-overview.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้อย่างลึกซึ้งว่าทำไม **RAII (Resource Acquisition Is Initialization)** ถึงเป็น
   หัวใจของภาษา C++ ที่ทำให้การจัดการทรัพยากรปลอดภัยกว่าภาษาอื่นที่ต้องพึ่ง garbage collector
   หรือ `try`/`finally`
2. เข้าใจว่า RAII ไม่ได้ใช้ได้แค่กับ memory เท่านั้น แต่ครอบคลุมทรัพยากรทุกชนิดที่มี "เปิด" และ
   "ปิด" เช่น file handle, mutex lock, socket connection
3. ใช้ `std::lock_guard` และ `std::unique_lock` จัดการ mutex แบบ RAII แทนการ
   `lock()`/`unlock()` ด้วยมือ พร้อมเข้าใจข้อแตกต่างของทั้งสองคลาส
4. เขียน **RAII wrapper class ของตัวเอง** สำหรับทรัพยากรที่ไม่มี smart pointer มาตรฐานให้ เช่น
   `FILE*`, POSIX mutex (`pthread_mutex_t`), semaphore (`sem_t`), และ socket file descriptor
   จาก Module C
5. เข้าใจแนวคิด **Rule of Zero** และอธิบายได้ว่าทำไม class ที่ประกอบด้วย RAII member ทั้งหมด
   ไม่จำเป็นต้องเขียน destructor, copy, หรือ move เองเลยสักตัว
6. เปรียบเทียบ **Rule of Zero**, **Rule of Three** (Part 46, 51), และ **Rule of Five** (จะ
   เจาะลึกเต็มรูปแบบใน Part 70) ได้อย่างชัดเจนว่าแต่ละกฎใช้เมื่อไหร่
7. จำแนกระดับของ **Exception Safety Guarantee** ทั้งสามระดับ (basic, strong, nothrow) และ
   ระบุได้ว่าโค้ดที่กำหนดให้มีระดับการรับประกันแบบใด
8. นำหลักการ RAII ทั้งหมดมาประกอบเป็น **checklist ระดับ production** สำหรับออกแบบการจัดการ
   ทรัพยากรในโปรเจกต์จริง

---

## 68.1 ทบทวนและขยายความ RAII อย่างจริงจัง: หัวใจของ C++ (Step 537)

เราพบกับ RAII มาแล้วสองครั้งก่อนหน้านี้ในหลักสูตร:

- **Part 46** แนะนำแนวคิด RAII เป็นครั้งแรกผ่านตัวอย่าง `SafeIntArray` ที่ผูก "การจัดหา
  ทรัพยากร (Resource Acquisition)" ไว้กับ constructor และ "การคืนทรัพยากร" ไว้กับ destructor
- **Part 54** ขยายความให้เห็นว่า RAII คือกลไกที่ทำให้ **Exception Safety** เป็นไปได้จริง ผ่าน
  แนวคิด **Stack Unwinding** — เมื่อเกิด exception, destructor ของทุก local object จะถูกเรียก
  โดยอัตโนมัติเสมอระหว่างที่ control flow "ปีน" กลับไปหา `catch` ที่ตรงชนิด
- **Part 67** แสดงให้เห็นว่า `unique_ptr`, `shared_ptr`, `weak_ptr` ทั้งหมดคือ **การประยุกต์ใช้
  RAII กับหน่วยความจำโดยเฉพาะ** — พวกมันไม่ใช่ "feature พิเศษ" ของภาษา แต่เป็นแค่ class ธรรมดา
  ที่เขียนตามหลัก RAII แล้วห่อ raw pointer ไว้ข้างใน

Part นี้จะรวบยอดทั้งหมดเข้าด้วยกันและขยายให้กว้างที่สุด: **RAII ไม่ใช่แค่เทคนิคหนึ่งในหลายๆ
เทคนิคของ C++ แต่มันคือปรัชญาการออกแบบที่ซึมอยู่ในทุกส่วนของภาษา** ตั้งแต่ `std::string`,
`std::vector`, `std::fstream`, `std::thread`, ไปจนถึง `std::mutex` — เกือบทุก class ใน
Standard Library เขียนตามหลัก RAII ทั้งสิ้น

### ทำไม RAII ถึงเหนือกว่าวิธีอื่นในการจัดการทรัพยากร

ลองเปรียบเทียบวิธีจัดการทรัพยากรในภาษาต่างๆ ที่อาจเคยได้ยินมา:

| แนวทาง | ภาษาที่ใช้ | ข้อเสีย |
|---|---|---|
| Garbage Collector (GC) | Java, C#, Python, Go | จัดการได้แค่ memory เท่านั้น ไม่รู้จัก "ปิดไฟล์" หรือ "ปลด lock" ให้ทันเวลาที่ต้องการ (ต้องพึ่ง `finally`/`using`/`with` เพิ่มอยู่ดี), เวลาที่ทรัพยากรถูกคืนไม่แน่นอน (ขึ้นกับรอบของ GC) |
| `try` / `finally` (หรือ `using`) | Java, C#, Python | ต้องเขียนซ้ำทุกจุดที่มีทรัพยากร, ลืมเขียนได้ง่าย, ซ้อนกันหลายชั้นจะอ่านยากมาก |
| Manual cleanup (`free`, `close`, `unlock`) | C | ต้องจำเองทุกจุด, ลืมง่าย, ไปไม่ถึงเมื่อเกิด error กลางทาง (ทบทวน Part 16) |
| **RAII** | **C++** | ผูกทรัพยากรกับ **อายุของ object (object lifetime)** โดยตรง — compiler เป็นคนบังคับให้ destructor ทำงานเสมอเมื่อ object หมด scope ไม่ว่าจะออกจาก scope แบบไหนก็ตาม (ปกติ, early return, break, exception) |

จุดที่ทำให้ RAII แข็งแกร่งกว่าคือ **มันไม่ใช่วินัยของโปรแกรมเมอร์ แต่เป็นกฎของภาษาที่
compiler บังคับใช้** — เราไม่สามารถ "ลืม" เรียก destructor ได้เลย เพราะ compiler เป็นคนแทรก
โค้ดเรียกให้เองที่จุดจบของทุก scope โดยอัตโนมัติ ต่างจาก `try`/`finally` ที่ต้องอาศัยให้
โปรแกรมเมอร์เขียน `finally` ครบทุกจุดด้วยตัวเอง

### RAII ในหนึ่งประโยค

> **RAII**: ผูกการ "จัดหาทรัพยากร (Acquisition)" ไว้กับ **constructor** และผูกการ "คืน
> ทรัพยากร (Release)" ไว้กับ **destructor** เสมอ — เมื่อ object ถูกสร้าง แปลว่าทรัพยากรพร้อม
> ใช้แล้ว เมื่อ object หมดอายุ (ไม่ว่าด้วยวิธีใด) ทรัพยากรจะถูกคืนโดยอัตโนมัติเสมอ

Part ที่เหลือของบทเรียนนี้จะแสดงให้เห็นว่าแนวคิดประโยคเดียวนี้นำไปประยุกต์ใช้กับทรัพยากรที่
หลากหลายกว่าแค่ memory ได้อย่างไรบ้าง

---

## 68.2 RAII เกินกว่า Memory: File Handle Wrapper (Step 538)

ทรัพยากรที่พบบ่อยที่สุดรองจาก memory คือ **file handle** — ในภาษา C เราใช้ `FILE*` จาก
`<cstdio>` ซึ่งต้อง `fclose()` เองเสมอ (ทบทวน Part 13) ถ้าลืม หรือมี exception เกิดขึ้นก่อนถึง
บรรทัด `fclose()` ไฟล์จะค้างเปิดอยู่จนกว่าโปรแกรมจะจบการทำงาน (หรือแย่กว่านั้นคือ file
descriptor ของระบบปฏิบัติการรั่วไหลจนถึง limit และโปรแกรมเปิดไฟล์ใหม่ไม่ได้อีกเลย)

C++ มีทางแก้ที่ "ฟรี" อยู่แล้วคือ `std::fstream`/`std::ifstream`/`std::ofstream` ซึ่งเป็น RAII
wrapper รอบไฟล์ที่ Standard Library เตรียมไว้ให้ (destructor ของมันเรียก `close()` อัตโนมัติ)
แต่บางครั้งเราจำเป็นต้องทำงานกับ C API เดิม (`FILE*`) โดยตรง เช่น เมื่อต้องเรียกใช้ไลบรารี C
ภายนอกที่รับ/คืนค่าเป็น `FILE*` เท่านั้น — กรณีนี้เราต้องเขียน **RAII wrapper class ของตัวเอง**

```cpp
// 01_file_raii.cpp - RAII wrapper รอบ FILE* ของตัวเอง (ไม่พึ่ง unique_ptr)
#include <cstdio>
#include <iostream>
#include <stdexcept>
#include <string>
#include <utility>

class FileGuard {
public:
    FileGuard(const std::string& path, const std::string& mode) {
        fp_ = std::fopen(path.c_str(), mode.c_str());
        if (!fp_) {
            throw std::runtime_error("เปิดไฟล์ไม่สำเร็จ: " + path);
        }
        std::cout << "  [เปิดไฟล์] " << path << '\n';
    }

    ~FileGuard() {
        if (fp_) {
            std::cout << "  [ปิดไฟล์อัตโนมัติ]\n";
            std::fclose(fp_);
        }
    }

    // ทำตาม Rule of Five เพราะ class นี้ถือ raw resource (FILE*) เอง:
    FileGuard(const FileGuard&) = delete;            // ห้าม copy (จะเกิด double-close)
    FileGuard& operator=(const FileGuard&) = delete;

    FileGuard(FileGuard&& other) noexcept : fp_(other.fp_) {
        other.fp_ = nullptr; // เจ้าของใหม่รับช่วงต่อ ตัวเก่าไม่ต้อง close
    }
    FileGuard& operator=(FileGuard&& other) noexcept {
        if (this != &other) {
            if (fp_) std::fclose(fp_);
            fp_ = other.fp_;
            other.fp_ = nullptr;
        }
        return *this;
    }

    void write(const std::string& text) {
        std::fputs(text.c_str(), fp_);
    }

    std::FILE* handle() const { return fp_; }

private:
    std::FILE* fp_ = nullptr;
};

int main() {
    const std::string path = "/tmp/raii_file_demo.txt";
    {
        FileGuard f(path, "w");
        f.write("hello RAII\n");
    } // ปิดไฟล์อัตโนมัติที่นี่

    {
        FileGuard f(path, "r");
        char buf[64] = {0};
        std::fgets(buf, sizeof(buf), f.handle());
        std::cout << "  อ่านได้: " << buf;
    }

    std::remove(path.c_str());

    try {
        FileGuard bad("/no/such/dir/file.txt", "r");
    } catch (const std::exception& e) {
        std::cout << "จับ error ได้: " << e.what() << '\n';
    }

    return 0;
}
```

ผลลัพธ์:

```
  [เปิดไฟล์] /tmp/raii_file_demo.txt
  [ปิดไฟล์อัตโนมัติ]
  [เปิดไฟล์] /tmp/raii_file_demo.txt
  อ่านได้: hello RAII
  [ปิดไฟล์อัตโนมัติ]
จับ error ได้: เปิดไฟล์ไม่สำเร็จ: /no/such/dir/file.txt
```

ตรวจสอบด้วย `valgrind --leak-check=full` แล้วไม่มี resource รั่วไหลเลย สังเกตประเด็นสำคัญ
สามข้อในการออกแบบ `FileGuard`:

1. **Constructor throw ได้ ถ้าทรัพยากรจัดหาไม่สำเร็จ** — นี่คือหลักการสำคัญของ RAII: ถ้า
   `fopen()` ล้มเหลว object ไม่ควรถูกสร้างขึ้นมาสำเร็จเลย (throw ออกจาก constructor) เพื่อไม่
   ให้มี `FileGuard` ที่ "ดูเหมือนใช้ได้" แต่จริงๆ ไม่มีไฟล์อยู่ข้างในเดินไปทั่วโปรแกรม
2. **ห้าม copy (deleted)** — ถ้า copy ได้ทั้งสอง object จะถือ `FILE*` เดียวกัน แล้วทั้งคู่ก็จะ
   `fclose()` มันตอนหมด scope กลายเป็น double-close ทันที (เทียบเท่า double free ของ memory)
3. **Move ได้ (custom move constructor/assignment)** — เพื่อให้ `FileGuard` ย้าย ownership
   ของไฟล์ระหว่างฟังก์ชันได้เหมือน `unique_ptr` (ต้องตั้งค่า `fp_` ของตัวต้นทางเป็น `nullptr`
   หลัง move เสมอ ไม่งั้นจะ `fclose()` ซ้ำตอนตัวต้นทางหมด scope)

สังเกตว่าโครงสร้างของ `FileGuard` แทบจะเหมือนกับสิ่งที่ `unique_ptr<FILE, FileCloser>` (custom
deleter ที่เรียนใน Part 67 หัวข้อ 67.7) ทำให้อยู่แล้ว — นี่คือสองทางเลือกที่ให้ผลลัพธ์เดียวกัน:
**ใช้ `unique_ptr` + custom deleter เมื่อ logic เพิ่มเติมมีน้อย** (แค่เปิด/ปิด) หรือ **เขียน
RAII class ของตัวเองเมื่อมี logic เฉพาะทางเพิ่มเติม** (เช่น method `write()` ที่ทำให้ API
ใช้งานง่ายและปลอดภัยกว่าเรียก `std::fputs` ตรงๆ)

---

## 68.3 RAII สำหรับ Mutex: std::lock_guard และ std::unique_lock (Step 539)

ใน **Part 32** เราเรียนรู้การใช้ `pthread_mutex_t` ในภาษา C ซึ่งต้อง `pthread_mutex_lock()`
และ `pthread_mutex_unlock()` เองทุกครั้ง — และเจอกับดักที่พบบ่อยที่สุดคือ **"ลืม unlock ตอน
early return"** ซึ่งทำให้ thread อื่นติดค้าง (deadlock) รอ mutex ที่ไม่มีวันถูกปลดล็อกตลอดไป
C++ Standard Library แก้ปัญหานี้ด้วย `std::mutex` (จาก `<mutex>`) ร่วมกับ RAII wrapper สอง
ตัวคือ `std::lock_guard` และ `std::unique_lock`

```cpp
// 02_lock_guard.cpp
#include <iostream>
#include <mutex>
#include <stdexcept>
#include <thread>
#include <vector>

std::mutex counterMutex;
long counter = 0;

void incrementManyTimes(int times) {
    for (int i = 0; i < times; ++i) {
        std::lock_guard<std::mutex> guard(counterMutex); // ล็อกทันทีที่สร้าง
        ++counter;
    } // unlock อัตโนมัติทุกรอบเมื่อ guard หมด scope แม้จะวนซ้ำหลายพันครั้ง
}

void riskyOperation(bool shouldThrow) {
    std::unique_lock<std::mutex> lock(counterMutex); // ล็อกเหมือน lock_guard
    ++counter;
    if (shouldThrow) {
        throw std::runtime_error("เกิดปัญหาระหว่างถือ lock อยู่");
        // lock จะถูกปลดล็อกอัตโนมัติระหว่าง stack unwinding แม้ throw ตรงนี้
    }
    lock.unlock();          // unique_lock ปลดล็อกเองก่อนหมด scope ได้ (lock_guard ทำไม่ได้)
    std::cout << "  ทำงานส่วนที่ไม่ต้องล็อกต่อ...\n";
    lock.lock();             // แล้วล็อกกลับใหม่ได้อีกถ้าจำเป็น
    std::cout << "  counter ปัจจุบัน = " << counter << '\n';
}

int main() {
    std::vector<std::thread> workers;
    for (int i = 0; i < 4; ++i) {
        workers.emplace_back(incrementManyTimes, 10000);
    }
    for (auto& t : workers) t.join();
    std::cout << "counter หลังรัน 4 threads x 10000 ครั้ง = " << counter << '\n';

    try {
        riskyOperation(true);
    } catch (const std::exception& e) {
        std::cout << "จับ error ได้: " << e.what() << " (counter = " << counter << ")\n";
    }

    riskyOperation(false);

    return 0;
}
```

คอมไพล์ (ต้อง link `-lpthread` บน Linux):

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 02_lock_guard.cpp -o 02_lg -lpthread
./02_lg
```

ผลลัพธ์:

```
counter หลังรัน 4 threads x 10000 ครั้ง = 40000
จับ error ได้: เกิดปัญหาระหว่างถือ lock อยู่ (counter = 40001)
  ทำงานส่วนที่ไม่ต้องล็อกต่อ...
  counter ปัจจุบัน = 40002
```

`counter` ได้ค่า **40000 พอดี** จากการรัน 4 threads x 10000 ครั้ง (ไม่มี race condition เลย
ทบทวน Part 32) และสังเกตว่าตอน `riskyOperation(true)` throw exception ออกมาขณะที่ยังถือ
`lock` อยู่ **โปรแกรมไม่ deadlock** — เพราะ `unique_lock` เป็น RAII object ที่ destructor ของมัน
จะปลดล็อกให้อัตโนมัติระหว่าง stack unwinding เหมือนกับ `FileHandle` ในตัวอย่างของ Part 54.6
ทุกประการ

### เปรียบเทียบ std::lock_guard กับ std::unique_lock

| คุณสมบัติ | `std::lock_guard` | `std::unique_lock` |
|---|---|---|
| Lock ทันทีที่สร้าง | ใช่ | ใช่ (ค่าเริ่มต้น — เลื่อน lock ออกไปก็ได้ด้วย `std::defer_lock`) |
| Unlock เองก่อนหมด scope ได้ (`unlock()`) | **ไม่ได้** | **ได้** — และ `lock()` กลับใหม่ได้ด้วย |
| ใช้กับ `std::condition_variable` ได้ | ไม่ได้ | **ได้** (condition variable ต้องการ `unique_lock` เท่านั้น — จะเจาะลึกใน Part 81-82) |
| Move ได้ | ไม่ได้ | ได้ |
| Overhead | ต่ำที่สุด (แทบไม่มีค่าใช้จ่ายเพิ่มจาก `mutex` ตรงๆ) | สูงกว่าเล็กน้อย (ต้องเก็บ state เพิ่มว่ากำลังถือ lock อยู่หรือไม่) |

> **กฎทอง**: ใช้ `std::lock_guard` เป็นค่าเริ่มต้นเสมอเมื่อแค่ต้องการ "ล็อกตลอด scope"
> เปลี่ยนไปใช้ `std::unique_lock` เฉพาะเมื่อต้องการความยืดหยุ่นเพิ่มเติม เช่น ปลดล็อกชั่วคราว
> กลาง scope, หรือต้องใช้กับ `std::condition_variable`

### เทียบกับ pthread_mutex_t แบบ manual จาก Part 32

| สถานการณ์ | `pthread_mutex_t` (Module C) | `std::lock_guard`/`std::unique_lock` (C++) |
|---|---|---|
| ทำงานสำเร็จปกติ | ต้องเขียน `pthread_mutex_unlock()` เองท้าย critical section | ปลดล็อกอัตโนมัติเมื่อ guard หมด scope |
| Early return กลาง critical section | **ลืม unlock ได้ง่ายมาก** (กับดักที่พบบ่อยที่สุดใน Part 32) | ปลดล็อกอัตโนมัติไม่ว่าจะ return ตรงไหน |
| เกิด exception ระหว่างถือ lock | ภาษา C ไม่มี exception จึงไม่เจอปัญหานี้โดยตรง แต่ถ้าเรียกจาก C++ ที่ throw ได้ จะ deadlock ทันที | ปลดล็อกอัตโนมัติระหว่าง stack unwinding เสมอ |

---

## 68.4 เขียน RAII Wrapper ของตัวเองสำหรับทรัพยากรที่ไม่มี Smart Pointer ให้ (Step 540)

`std::lock_guard` ใช้ได้กับ `std::mutex` ของ C++ เท่านั้น แต่ในโปรแกรมที่ผสมกันระหว่างโค้ด C
เดิม (Module C) กับ C++ ใหม่ เรามักต้องทำงานกับทรัพยากรของ POSIX โดยตรง เช่น `pthread_mutex_t`,
`sem_t`, หรือ socket file descriptor (`int`) — ทรัพยากรเหล่านี้ **ไม่มี RAII wrapper มาตรฐาน
ให้ใช้** จึงต้องเขียนเอง โดยยึดหลักการเดียวกับ `FileGuard` ในหัวข้อก่อนหน้า

### ตัวอย่างที่ 1: ห่อ pthread_mutex_t ด้วย RAII สองชั้น

แนวทางที่แนะนำคือแยกเป็น **สอง class**: class แรกห่อ "ตัวทรัพยากร mutex เอง" (แทนที่
`pthread_mutex_init`/`pthread_mutex_destroy` แบบ manual) ส่วน class ที่สองห่อ "การ lock/unlock"
ให้ผูกกับ scope (ทำหน้าที่เหมือน `std::lock_guard` แต่สำหรับ mutex ของเราเอง):

```cpp
// 03_posix_mutex_raii.cpp - ห่อ pthread_mutex_t (Module C) ด้วย RAII เอง
#include <iostream>
#include <pthread.h>
#include <stdexcept>

// ---------- คลาสที่ 1: ห่อ "ทรัพยากร mutex" เอง (แทน pthread_mutex_init/destroy manual) ----------
class PosixMutex {
public:
    PosixMutex() {
        if (pthread_mutex_init(&mutex_, nullptr) != 0) {
            throw std::runtime_error("pthread_mutex_init ล้มเหลว");
        }
    }
    ~PosixMutex() {
        pthread_mutex_destroy(&mutex_);
    }

    // ทรัพยากร OS แบบนี้ copy ไม่ได้โดยธรรมชาติ (มันคือ handle เดียวของระบบปฏิบัติการ)
    PosixMutex(const PosixMutex&) = delete;
    PosixMutex& operator=(const PosixMutex&) = delete;
    PosixMutex(PosixMutex&&) = delete;
    PosixMutex& operator=(PosixMutex&&) = delete;

    void lock() { pthread_mutex_lock(&mutex_); }
    void unlock() { pthread_mutex_unlock(&mutex_); }

private:
    pthread_mutex_t mutex_{};
};

// ---------- คลาสที่ 2: ห่อ "การ lock/unlock" ให้ผูกกับ scope (เหมือน std::lock_guard) ----------
class PosixLockGuard {
public:
    explicit PosixLockGuard(PosixMutex& m) : mutex_(m) {
        mutex_.lock();
    }
    ~PosixLockGuard() {
        mutex_.unlock();
    }
    PosixLockGuard(const PosixLockGuard&) = delete;
    PosixLockGuard& operator=(const PosixLockGuard&) = delete;

private:
    PosixMutex& mutex_;
};

PosixMutex g_mutex;
long g_counter = 0;

void criticalSection() {
    PosixLockGuard guard(g_mutex); // lock เมื่อสร้าง
    ++g_counter;
    std::cout << "  g_counter = " << g_counter << '\n';
} // unlock อัตโนมัติเมื่อ guard หมด scope ไม่ว่าจะออกทางไหน

int main() {
    for (int i = 0; i < 3; ++i) {
        criticalSection();
    }
    std::cout << "จบการทำงาน g_counter สุดท้าย = " << g_counter << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 03_posix_mutex_raii.cpp -o 03_pm -lpthread
./03_pm
```

```
  g_counter = 1
  g_counter = 2
  g_counter = 3
จบการทำงาน g_counter สุดท้าย = 3
```

การแยกเป็นสอง class แบบนี้ (บางครั้งเรียกว่า **Resource class** กับ **Guard class**) เป็นสำนวน
ที่พบบ่อยมากในโค้ด production เพราะแยกความรับผิดชอบชัดเจน: `PosixMutex` รับผิดชอบแค่ "ทรัพยากร
มีอยู่จริงและถูกทำลายถูกต้อง" ส่วน `PosixLockGuard` รับผิดชอบแค่ "การใช้งานทรัพยากรนั้นผูกกับ
scope ปัจจุบัน" — นี่คือโครงสร้างเดียวกับที่ `std::mutex` (Resource) กับ `std::lock_guard`
(Guard) ใน Standard Library ใช้จริง

### ตัวอย่างที่ 2: ห่อ Socket File Descriptor ด้วย RAII

ทบทวนจาก **Part 33-34**: socket ในภาษา C คือแค่ตัวเลข `int` (file descriptor) ที่ต้อง
`close()` เองเสมอ ลืม `close()` แม้แต่ที่เดียวก็ทำให้ file descriptor รั่วไหลจนกระทบทั้งระบบได้
(OS มี limit จำนวน file descriptor ต่อโปรเซส)

```cpp
// 04_socket_raii.cpp - RAII wrapper รอบ socket file descriptor (ทบทวน Part 33-34)
#include <iostream>
#include <stdexcept>
#include <sys/socket.h>
#include <unistd.h>
#include <utility>

class SocketGuard {
public:
    SocketGuard(int domain, int type, int protocol) {
        fd_ = ::socket(domain, type, protocol);
        if (fd_ < 0) {
            throw std::runtime_error("สร้าง socket ไม่สำเร็จ");
        }
        std::cout << "  [เปิด socket] fd = " << fd_ << '\n';
    }

    ~SocketGuard() {
        if (fd_ >= 0) {
            std::cout << "  [ปิด socket อัตโนมัติ] fd = " << fd_ << '\n';
            ::close(fd_);
        }
    }

    SocketGuard(const SocketGuard&) = delete;
    SocketGuard& operator=(const SocketGuard&) = delete;

    SocketGuard(SocketGuard&& other) noexcept : fd_(std::exchange(other.fd_, -1)) {}
    SocketGuard& operator=(SocketGuard&& other) noexcept {
        if (this != &other) {
            if (fd_ >= 0) ::close(fd_);
            fd_ = std::exchange(other.fd_, -1);
        }
        return *this;
    }

    int get() const { return fd_; }

private:
    int fd_ = -1;
};

int main() {
    {
        SocketGuard sock(AF_INET, SOCK_STREAM, 0);
        std::cout << "  ใช้งาน socket fd = " << sock.get() << " ต่อได้ตามปกติ\n";
    } // ปิด socket อัตโนมัติที่นี่ แม้จะลืมเขียน close() ก็ไม่รั่วไหล

    std::cout << "จบ main\n";
    return 0;
}
```

```
  [เปิด socket] fd = 3
  ใช้งาน socket fd = 3 ต่อได้ตามปกติ
  [ปิด socket อัตโนมัติ] fd = 3
จบ main
```

สังเกตการใช้ `std::exchange` (จาก `<utility>`) ใน move constructor/assignment — มันทำสองอย่าง
พร้อมกันในบรรทัดเดียว: **อ่านค่าเดิมของ `other.fd_` ออกมา** และ **ตั้งค่าใหม่ให้ `other.fd_`
เป็น `-1`** ซึ่งเป็นสำนวนที่ปลอดภัยและกระชับกว่าการเขียนแยกสองบรรทัด (`int old = other.fd_;
other.fd_ = -1;`) และเป็นสำนวนมาตรฐานที่ใช้เขียน move operation ของ RAII class แทบทุกที่ใน
โค้ด C++ สมัยใหม่

> **สังเกตรูปแบบร่วมของทั้งสามตัวอย่าง (`FileGuard`, `PosixMutex`, `SocketGuard`)**: ทั้งหมด
> ทำตามพิมพ์เขียวเดียวกันทุกประการ — (1) constructor จัดหาทรัพยากรและ throw ถ้าล้มเหลว (2)
> destructor คืนทรัพยากรเสมอ (3) ห้าม copy เพราะทรัพยากรมีเจ้าของเดียว (4) อนุญาตให้ move ได้
> ถ้าต้องการย้าย ownership ระหว่างฟังก์ชัน นี่คือ **พิมพ์เขียวมาตรฐานของ RAII wrapper class**
> ที่ใช้ได้กับทรัพยากรแทบทุกชนิดในโลกที่มี "เปิด" และ "ปิด"

---

## 68.5 Rule of Zero (Step 541)

หลังจากเขียน RAII wrapper class มาหลายตัวที่ต้องกำหนด destructor, copy, และ move เองครบทุก
ฟังก์ชัน (**Rule of Five** ที่เกริ่นไว้ใน Part 46 และ Part 51) คำถามที่ตามมาคือ: "แล้วถ้า class
ของเราแค่ **ประกอบด้วย** RAII object อื่นๆ (เช่น `std::string`, `std::vector`, `unique_ptr`,
`shared_ptr`) โดยไม่ได้ถือ raw resource เองเลย จำเป็นต้องเขียน destructor/copy/move เองไหม?"

คำตอบคือ **ไม่จำเป็นเลย** — และนี่คือแก่นของ **Rule of Zero**:

> **Rule of Zero**: ถ้า class ของคุณไม่ได้ถือ raw resource ใดๆ โดยตรง (raw pointer, raw file
> handle, raw OS handle) แต่ประกอบด้วย **สมาชิกที่เป็น RAII type ทั้งหมด** ให้ **ไม่ต้องเขียน**
> destructor, copy constructor, copy assignment, move constructor, หรือ move assignment เอง
> เลยแม้แต่ตัวเดียว ปล่อยให้ compiler generate ให้ทั้งหมดโดยอัตโนมัติ

เหตุผลคือ compiler-generated destructor จะเรียก destructor ของสมาชิกทุกตัวให้อัตโนมัติอยู่แล้ว
(เรียงตามลำดับย้อนกลับของการประกาศ) เช่นเดียวกับ compiler-generated copy/move constructor ที่
จะ copy/move สมาชิกแต่ละตัวให้ถูกต้องตามชนิดของมันเองอยู่แล้ว — ถ้าสมาชิกทุกตัวจัดการตัวเองได้
ถูกต้องสมบูรณ์แบบอยู่แล้ว (เพราะเป็น RAII type) การเขียน destructor/copy/move ทับซ้ำเข้าไปเอง
มีแต่จะเพิ่มโอกาสเกิดบั๊ก (เขียนผิด, ลืม field ใหม่ที่เพิ่มเข้ามาทีหลัง) โดยไม่ได้ประโยชน์อะไร
เพิ่มเลย

```cpp
// 05_rule_of_zero.cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

// Report ไม่ต้องเขียน destructor / copy / move เองเลยสักตัว (Rule of Zero)
// เพราะสมาชิกทุกตัวเป็น RAII type ที่จัดการตัวเองอยู่แล้ว: string, vector, shared_ptr
class Report {
public:
    Report(std::string title, std::vector<int> scores)
        : title_(std::move(title)),
          scores_(std::move(scores)),
          note_(std::make_shared<std::string>("draft")) {}

    void print() const {
        std::cout << "  รายงาน: " << title_ << " (note=" << *note_ << ") คะแนน: ";
        for (int s : scores_) std::cout << s << ' ';
        std::cout << '\n';
    }

    // ไม่มี ~Report(), ไม่มี copy/move constructor หรือ assignment ที่เขียนเอง
    // compiler จะ generate ให้ครบทั้งหมดโดยอัตโนมัติ และทำงานถูกต้องเสมอ เพราะ
    // string, vector, shared_ptr ต่างก็มี copy/move/destructor ของตัวเองที่ถูกต้องอยู่แล้ว

private:
    std::string title_;
    std::vector<int> scores_;
    std::shared_ptr<std::string> note_;
};

int main() {
    Report r1("รายงานไตรมาส 1", {80, 90, 75});
    r1.print();

    Report r2 = std::move(r1); // compiler-generated move constructor ทำงานถูกต้อง
    r2.print();

    Report r3("รายงานไตรมาส 2", {60, 70});
    r3 = r2; // ไม่ได้ move -> compiler-generated copy assignment (copy string/vector, ใช้ shared_ptr ร่วมกัน)
    r3.print();

    return 0;
}
```

ผลลัพธ์:

```
  รายงาน: รายงานไตรมาส 1 (note=draft) คะแนน: 80 90 75
  รายงาน: รายงานไตรมาส 1 (note=draft) คะแนน: 80 90 75
  รายงาน: รายงานไตรมาส 1 (note=draft) คะแนน: 80 90 75
```

`Report` **ไม่มีบรรทัดใดเลย** ที่เขียน `~Report()`, `Report(const Report&)`, หรือ
`operator=` เอง แต่ทุกอย่างทำงานถูกต้องสมบูรณ์: การ `move` ย้าย `title_`/`scores_`/`note_`
ให้ถูกต้องตามชนิดของมันเอง (string/vector ถูกย้ายแบบ move จริง ไม่ copy) และการ copy assignment
ก็ copy `title_`/`scores_` แบบ deep copy ตามธรรมชาติของ `string`/`vector` ในขณะที่ `note_`
(เป็น `shared_ptr`) จะกลายเป็นการแชร์ตัวชี้เดียวกัน (เพิ่ม `use_count`) ตามธรรมชาติของ
`shared_ptr` — ทั้งหมดนี้ compiler จัดการให้ถูกต้องโดยที่เราไม่ต้องเขียนโค้ดเพิ่มแม้แต่บรรทัด
เดียว

### Rule of Zero ในทางปฏิบัติ: นี่คือเหตุผลที่ Part 55 ทำงานได้โดยไม่มี delete เลย

ย้อนกลับไปที่ **Part 55** — คลาส `Library` เก็บ `vector<unique_ptr<Media>>` และ
`vector<unique_ptr<Member>>` เป็นสมาชิก โดยไม่เคยเขียน destructor ของ `Library` เองเลยแม้แต่
บรรทัดเดียว นี่คือ **Rule of Zero ที่ถูกใช้งานจริงมาตั้งแต่ Part 55** โดยที่ตอนนั้นยังไม่ได้
เรียกชื่อมันอย่างเป็นทางการ — เพราะสมาชิกทั้งสองตัวเป็น RAII type (`vector` ของ `unique_ptr`)
ที่จัดการตัวเองได้สมบูรณ์อยู่แล้ว `Library` จึงไม่ต้องทำอะไรเพิ่มเติมเลย

---

## 68.6 Rule of Zero เทียบกับ Rule of Three และ Rule of Five (Step 542)

ทบทวนสั้นๆ: **Rule of Three** (Part 46, 51) กล่าวว่าถ้า class ต้องเขียน destructor เอง (เพราะ
ถือ raw resource) ก็มักจะต้องเขียน copy constructor และ copy assignment operator เองด้วย
เพราะ default (compiler-generated) copy จะแค่ copy ค่า pointer ตรงๆ (shallow copy) ทำให้เกิด
double free เมื่อทั้งสอง object พยายามคืนทรัพยากรเดียวกัน ส่วน **Rule of Five** (จะเจาะลึกเต็ม
รูปแบบใน **Part 70** เมื่อเรียนเรื่อง Move Semantics) คือ Rule of Three ฉบับขยายที่เพิ่ม move
constructor และ move assignment operator เข้ามาด้วย เพื่อให้ resource ย้าย ownership ได้อย่าง
มีประสิทธิภาพแทนที่จะต้อง deep copy ทุกครั้ง

ตัวอย่างที่แสดงความแตกต่างชัดเจนที่สุดคือการเปรียบเทียบ class สองตัวที่ทำหน้าที่เหมือนกัน
ตัวหนึ่งถือ raw resource เอง (ต้อง Rule of Five) อีกตัวใช้ RAII type ของ STL แทน (Rule of
Zero พอ):

```cpp
// 06_rule_of_five_contrast.cpp - เมื่อไหร่ต้อง Rule of Five เทียบกับ Rule of Zero
#include <algorithm>
#include <iostream>
#include <utility>

// IntBuffer ถือ raw resource (int*) เอง -> ต้องทำตาม Rule of Five ครบทั้ง 5 ฟังก์ชัน
class IntBuffer {
public:
    explicit IntBuffer(std::size_t size) : size_(size), data_(new int[size]{}) {
        std::cout << "  [จอง buffer ขนาด " << size_ << "]\n";
    }
    ~IntBuffer() {
        std::cout << "  [คืน buffer]\n";
        delete[] data_;
    }
    IntBuffer(const IntBuffer& other) : size_(other.size_), data_(new int[other.size_]) {
        std::cout << "  [copy constructor: deep copy]\n";
        std::copy(other.data_, other.data_ + size_, data_);
    }
    IntBuffer& operator=(const IntBuffer& other) {
        std::cout << "  [copy assignment: deep copy]\n";
        if (this != &other) {
            int* newData = new int[other.size_];
            std::copy(other.data_, other.data_ + other.size_, newData);
            delete[] data_;
            data_ = newData;
            size_ = other.size_;
        }
        return *this;
    }
    IntBuffer(IntBuffer&& other) noexcept
        : size_(std::exchange(other.size_, 0)), data_(std::exchange(other.data_, nullptr)) {
        std::cout << "  [move constructor]\n";
    }
    IntBuffer& operator=(IntBuffer&& other) noexcept {
        std::cout << "  [move assignment]\n";
        if (this != &other) {
            delete[] data_;
            size_ = std::exchange(other.size_, 0);
            data_ = std::exchange(other.data_, nullptr);
        }
        return *this;
    }

    std::size_t size() const { return size_; }

private:
    std::size_t size_;
    int* data_;
};

// Wrapper ที่ใช้ RAII type ของ STL แทน -> Rule of Zero พอ ไม่ต้องเขียนอะไรเองเลย
#include <vector>
class SafeIntBuffer {
public:
    explicit SafeIntBuffer(std::size_t size) : data_(size) {}
    std::size_t size() const { return data_.size(); }

private:
    std::vector<int> data_; // vector จัดการ Rule of Five ให้ครบถ้วนอยู่แล้วภายใน
};

int main() {
    std::cout << "-- IntBuffer (ต้องเขียน Rule of Five เอง) --\n";
    IntBuffer a(4);
    IntBuffer b = a;             // copy constructor
    IntBuffer c = std::move(a);  // move constructor
    b = c;                       // copy assignment

    std::cout << "-- SafeIntBuffer (Rule of Zero) --\n";
    SafeIntBuffer sa(4);
    SafeIntBuffer sb = sa;            // compiler-generated copy (เรียก vector's copy ให้)
    SafeIntBuffer sc = std::move(sa); // compiler-generated move
    std::cout << "sb.size() = " << sb.size() << ", sc.size() = " << sc.size() << '\n';

    return 0;
}
```

ผลลัพธ์:

```
-- IntBuffer (ต้องเขียน Rule of Five เอง) --
  [จอง buffer ขนาด 4]
  [copy constructor: deep copy]
  [move constructor]
  [copy assignment: deep copy]
-- SafeIntBuffer (Rule of Zero) --
sb.size() = 4, sc.size() = 4
  [คืน buffer]
  [คืน buffer]
  [คืน buffer]
```

`IntBuffer` ต้องเขียนโค้ดรวม 5 ฟังก์ชันพิเศษ (destructor + copy 2 + move 2) รวมกว่า 30 บรรทัด
ในขณะที่ `SafeIntBuffer` ทำสิ่งเดียวกันทุกประการโดยเขียนแค่ constructor เดียว เพราะ
`std::vector<int>` ได้แก้ปัญหา Rule of Five ให้เราไว้เรียบร้อยแล้วข้างในตัวมันเอง (ตัว
`std::vector` เองก็ถือ raw pointer ภายในและต้องทำตาม Rule of Five เช่นกัน แต่นั่นเป็นความ
รับผิดชอบของทีม STL ไม่ใช่ของเรา)

### ตารางสรุป: เมื่อไหร่ใช้กฎไหน

| กฎ | ใช้เมื่อ | ตัวอย่าง |
|---|---|---|
| **Rule of Zero** | Class ไม่ถือ raw resource เอง สมาชิกทุกตัวเป็น RAII type (`string`, `vector`, smart pointer, ฯลฯ) | `Report`, `Library` (Part 55), เกือบทุก class ระดับ business logic ที่ไม่เกี่ยวกับ low-level resource โดยตรง |
| **Rule of Three** | Class ถือ raw resource เอง และยังไม่ได้ใช้ move semantics (หรือทำงานในโค้ดเบสที่ยังไม่ใช้ C++11 ขึ้นไป) | ตัวอย่าง `IntArray` ใน Part 51 |
| **Rule of Five** | Class ถือ raw resource เอง และต้องการรองรับ move semantics เพื่อประสิทธิภาพ (มาตรฐานตั้งแต่ C++11 เป็นต้นไป) | `IntBuffer`, `FileGuard`, `SocketGuard` ในบทเรียนนี้ |

> **กฎทองของการออกแบบ class ใน C++ สมัยใหม่**: **พยายามออกแบบให้ class ของคุณทำตาม Rule of
> Zero ได้เสมอ** โดยการ "ผลักภาระ" การจัดการ raw resource ไปให้ RAII wrapper ชั้นในสุด (เช่น
> `unique_ptr`, `vector`, หรือ RAII class ที่เขียนเองแบบ `FileGuard`/`SocketGuard`) รับผิดชอบ
> แทน แล้วให้ class ระดับสูงกว่าประกอบร่างจาก RAII type เหล่านั้นเท่านั้น การเขียน Rule of
> Five เองควรเกิดขึ้น **แค่ที่จุดเดียว** ในโค้ดเบส (จุดที่ห่อ raw resource เข้าเป็น RAII class
> เป็นครั้งแรก) ไม่ใช่กระจายอยู่ทั่วทุก class ในโปรแกรม

---

## 68.7 Exception Safety Guarantee Levels: Basic / Strong / Nothrow (Step 543)

ทบทวนจาก **Part 54.6**: RAII ทำให้ resource ไม่รั่วไหลเมื่อเกิด exception เสมอ แต่นั่นเป็นแค่
ส่วนหนึ่งของเรื่อง **Exception Safety** เท่านั้น คำถามที่ลึกกว่าคือ: "เมื่อฟังก์ชันหนึ่ง throw
exception ออกมากลางทาง **สถานะ (state)** ของ object ที่มันกำลังแก้ไขอยู่จะเป็นอย่างไร?" C++
Standard Library แบ่งระดับการรับประกันนี้ไว้ 3 ระดับ (เรียงจากอ่อนไปแก่):

### ระดับ 1: Basic Guarantee (การรับประกันขั้นพื้นฐาน)

รับประกันว่าถ้า operation ล้มเหลว (throw) object จะยังอยู่ใน**สถานะที่ใช้งานได้** (invariant
ยังคงอยู่ ไม่มี resource รั่วไหล ไม่มี object เสียหายจนใช้ต่อไม่ได้) **แต่ค่าข้างในอาจเปลี่ยน
ไปจากก่อนเรียก operation นั้น** (ไม่มีการ rollback กลับสถานะเดิม)

### ระดับ 2: Strong Guarantee (การรับประกันขั้นสูง / All-or-Nothing)

รับประกันว่าถ้า operation ล้มเหลว **สถานะของ object จะเหมือนเดิมทุกประการ** เสมือนไม่เคยเรียก
operation นั้นเลย (**all-or-nothing**) — ทำได้บ่อยที่สุดด้วยเทคนิคที่เรียกว่า **copy-and-swap
idiom**: เตรียมข้อมูลใหม่ทั้งหมดบน "สำเนา" ก่อน ถ้าขั้นตอนไหนล้มเหลว object เดิมจะยังไม่ถูก
แตะต้องเลย แล้วค่อย `swap()` (ซึ่งต้องเป็น `noexcept` เสมอ) เข้าแทนที่ในขั้นตอนสุดท้ายที่รับรอง
ว่าจะไม่มีทาง throw

### ระดับ 3: Nothrow Guarantee (การรับประกันว่าไม่มีทาง throw)

รับประกันว่า operation นี้ **ไม่มีทาง throw exception ออกมาเลย** ใช้กับ operation ที่จำเป็นต้อง
ปลอดภัย 100% เสมอ เช่น destructor (destructor ที่ throw เป็นสิ่งอันตรายมาก — ถ้าเกิดขึ้นระหว่าง
stack unwinding จาก exception อื่นที่กำลังเกิดอยู่แล้ว โปรแกรมจะเรียก `std::terminate()` ทันที)
`swap()`, และ move constructor/move assignment (ที่ต้องทำเครื่องหมาย `noexcept` เสมอเพื่อให้
`std::vector` ยอม "ย้าย" element แทน "copy" ตอน reallocate — ถ้า move ของเรา throw ได้
`std::vector` จะเลือก copy แทนเพื่อความปลอดภัย ทำให้เสียประสิทธิภาพฟรีๆ)

```cpp
// 07_exception_safety.cpp - Basic / Strong / Nothrow guarantee
#include <algorithm>
#include <iostream>
#include <stdexcept>
#include <vector>

// ---------- Nothrow guarantee ----------
class Point {
public:
    Point(double x, double y) noexcept : x_(x), y_(y) {}
    void swap(Point& other) noexcept {
        std::swap(x_, other.x_);
        std::swap(y_, other.y_);
    }
    double x() const noexcept { return x_; }
    double y() const noexcept { return y_; }

private:
    double x_, y_;
};

// ---------- Strong guarantee ----------
// ถ้า operation ล้มเหลว (throw) รับประกันว่า state ของ object จะ "เหมือนเดิมทุกประการ"
// เหมือนไม่มีอะไรเกิดขึ้น (all-or-nothing) — ทำได้ด้วย "copy-and-swap idiom"
class SafeCatalog {
public:
    void addTitle(const std::string& title) {
        if (title.empty()) {
            throw std::invalid_argument("title ห้ามว่าง");
        }
        // (1) เตรียมข้อมูลใหม่บน "สำเนา" ก่อน ถ้า throw ตรงนี้ this->titles_ ยังไม่ถูกแตะเลย
        std::vector<std::string> temp = titles_;
        temp.push_back(title);
        // (2) ถ้าถึงบรรทัดนี้ได้แปลว่าทุกอย่างสำเร็จแน่นอน -> swap แบบ noexcept
        titles_.swap(temp);
    }
    std::size_t size() const { return titles_.size(); }
    void print() const {
        for (const auto& t : titles_) std::cout << "[" << t << "] ";
        std::cout << '\n';
    }

private:
    std::vector<std::string> titles_;
};

// ---------- Basic guarantee ----------
// ถ้า operation ล้มเหลว รับประกันแค่ว่า object จะยังอยู่ใน "สถานะที่ใช้งานได้" (invariant คง
// อยู่) และไม่รั่วไหล resource แต่ "ค่าข้างในอาจเปลี่ยนไปจากก่อนเรียก"
class BasicCatalog {
public:
    void addTwoTitles(const std::string& first, const std::string& second) {
        titles_.push_back(first); // สำเร็จแล้ว แก้ไม่ได้
        if (second.empty()) {
            throw std::invalid_argument("title ที่สองห้ามว่าง");
            // titles_ ตอนนี้มี "first" อยู่แล้ว แม้ operation โดยรวมจะถือว่า "ล้มเหลว"
            // แต่ container ยังอยู่ในสถานะที่ valid (ไม่พัง ไม่ leak) จึงเป็นแค่ basic guarantee
        }
        titles_.push_back(second);
    }
    std::size_t size() const { return titles_.size(); }

private:
    std::vector<std::string> titles_;
};

int main() {
    std::cout << "-- Strong guarantee: SafeCatalog --\n";
    SafeCatalog safe;
    safe.addTitle("Effective C++");
    try {
        safe.addTitle(""); // จะ throw ก่อนแตะ titles_ เลย
    } catch (const std::exception& e) {
        std::cout << "จับ error ได้: " << e.what() << " (size ยังคงเป็น " << safe.size() << ")\n";
    }
    safe.print();

    std::cout << "-- Basic guarantee: BasicCatalog --\n";
    BasicCatalog basic;
    try {
        basic.addTwoTitles("A", "");
    } catch (const std::exception& e) {
        std::cout << "จับ error ได้: " << e.what()
                  << " (แต่ size กลายเป็น " << basic.size() << " ไปแล้ว ไม่ rollback)\n";
    }

    std::cout << "-- Nothrow guarantee: Point::swap --\n";
    Point p1(1.0, 2.0), p2(3.0, 4.0);
    p1.swap(p2); // ไม่มีทาง throw แน่นอน (noexcept)
    std::cout << "p1 = (" << p1.x() << ", " << p1.y() << ")\n";

    return 0;
}
```

ผลลัพธ์:

```
-- Strong guarantee: SafeCatalog --
จับ error ได้: title ห้ามว่าง (size ยังคงเป็น 1)
[Effective C++]
-- Basic guarantee: BasicCatalog --
จับ error ได้: title ที่สองห้ามว่าง (แต่ size กลายเป็น 1 ไปแล้ว ไม่ rollback)
-- Nothrow guarantee: Point::swap --
p1 = (3, 4)
```

สังเกตความแตกต่างชัดเจน: `SafeCatalog::addTitle("")` throw ออกมาโดยที่ `size()` ยังเป็น 1
เหมือนก่อนเรียก (**strong guarantee** — เหมือนไม่มีอะไรเกิดขึ้น) ในขณะที่
`BasicCatalog::addTwoTitles("A", "")` throw ออกมาแต่ `size()` กลายเป็น 1 ไปแล้ว (เพราะ `"A"`
ถูกเพิ่มไปสำเร็จก่อนที่จะพบว่า `second` ว่าง) — นี่คือ **basic guarantee**: object ยังใช้งานได้
ปกติ ไม่พัง ไม่ leak แต่ค่าข้างในเปลี่ยนไปแล้วบางส่วน

> **หลักปฏิบัติ**: ไม่ใช่ทุกฟังก์ชันจำเป็นต้องมี strong guarantee เสมอไป (บางครั้งมันแพงเกิน
> ไปในแง่ประสิทธิภาพ เพราะต้อง copy ข้อมูลชั่วคราวก่อนเสมอ) แต่**ทุกฟังก์ชันควรมีอย่างน้อย
> basic guarantee เสมอ** (ห้าม resource รั่วไหลหรือ object เสียหายจนใช้ต่อไม่ได้เด็ดขาด) ส่วน
> **destructor, swap, และ move operation ควรมี nothrow guarantee เสมอ** ไม่มีข้อยกเว้น — นี่
> คือกฎที่ **C++ Core Guidelines** (จะเจาะลึกใน Part 80) ย้ำหนักแน่นที่สุดข้อหนึ่ง

---

## 68.8 RAII Checklist ระดับ Production และแนวทางออกแบบ (Step 544)

รวบยอดทุกหลักการใน Part นี้เป็น checklist ที่ใช้ตรวจสอบได้จริงเวลาออกแบบหรือ review โค้ดที่
เกี่ยวกับการจัดการทรัพยากร:

### Checklist: ก่อนเขียน class ใหม่ที่เกี่ยวข้องกับทรัพยากร

- [ ] **ระบุให้ชัดว่า class นี้ "เป็นเจ้าของ" ทรัพยากรอะไรบ้าง** ถ้าไม่ได้เป็นเจ้าของอะไรเลย
      (แค่ยืมใช้) ไม่ต้องมี destructor ที่คืนอะไรเลย
- [ ] **ถ้าเป็นเจ้าของ raw resource โดยตรง** (raw pointer, `FILE*`, OS handle) ให้เขียนเป็น
      RAII wrapper ตามพิมพ์เขียวในหัวข้อ 68.4: constructor จัดหา + throw เมื่อล้มเหลว,
      destructor คืนเสมอ (`noexcept`), copy ตามสมควร (มักจะ `= delete`), move ตามสมควร
      (มักจะทำเสมอเพื่อประสิทธิภาพ)
- [ ] **ถ้าเป็นไปได้ ให้ใช้ RAII type ที่มีอยู่แล้วแทนการเขียนเอง**: `unique_ptr`/`shared_ptr`
      สำหรับ memory (Part 67), `std::fstream` สำหรับไฟล์, `std::lock_guard`/`std::unique_lock`
      สำหรับ mutex, `std::vector`/`std::string` แทน raw array — เขียน RAII wrapper เองเฉพาะ
      เมื่อไม่มีตัวเลือกสำเร็จรูปให้ใช้จริงๆ เท่านั้น
- [ ] **class ระดับสูงกว่าที่ประกอบด้วย RAII type เหล่านี้ ให้ยึด Rule of Zero เสมอ** — อย่า
      เขียน destructor/copy/move เองถ้าไม่จำเป็นจริงๆ
- [ ] **destructor, `swap()`, move constructor, move assignment ต้องเป็น `noexcept` เสมอ**
      (nothrow guarantee) — ตรวจสอบด้วย `static_assert(std::is_nothrow_move_constructible_v<T>)`
      ได้ถ้าต้องการความมั่นใจระดับ compile-time
- [ ] **ระบุระดับ exception safety guarantee ของแต่ละ public method อย่างชัดเจน** (เขียนไว้ใน
      comment หรือ documentation) อย่างน้อยต้องมี basic guarantee เสมอ ถ้าทำได้ให้พยายามไปถึง
      strong guarantee โดยเฉพาะ method ที่แก้ไขข้อมูลสำคัญ (เช่น การเงิน, transaction)
- [ ] **อย่าให้ resource รั่วไหลผ่านทาง exception ที่ throw จาก constructor**: ถ้า constructor
      ของ class จัดหาทรัพยากรหลายอย่างและอย่างที่สองล้มเหลว ต้องมั่นใจว่าอย่างแรกถูกคืนแล้ว
      (โดยทั่วไปทำได้อัตโนมัติถ้าใช้ RAII member — ถ้าสมาชิกตัวที่สองใน constructor throw
      สมาชิกตัวแรกที่สร้างเสร็จแล้วจะถูก destructor เรียกให้อัตโนมัติเสมอ)
- [ ] **ทดสอบด้วย valgrind หรือ AddressSanitizer เสมอ** (ทบทวน Part 38, จะเจาะลึก sanitizer
      เพิ่มเติมใน Part 96) โดยเฉพาะกับ code path ที่มี exception หรือ early return เกี่ยวข้อง
      เพราะเป็นจุดที่ resource มักรั่วไหลถ้าออกแบบ RAII ไม่ครบถ้วน

### ภาพรวมสุดท้าย: RAII คือด่านป้องกันที่ compiler บังคับใช้ให้ฟรี

ทบทวนภาพรวมทั้ง Module E ตอนท้าย: เราเริ่มจาก raw pointer + `new`/`delete` ที่เสี่ยงต่อทั้ง
memory leak, double free, และ dangling pointer (67.1) แก้ด้วย `unique_ptr`/`shared_ptr`/
`weak_ptr` (Part 67 ทั้ง Part) ซึ่งเป็นแค่การประยุกต์ใช้ RAII กับ memory โดยเฉพาะ จากนั้นขยาย
RAII ให้ครอบคลุมทรัพยากรทุกชนิด (file, mutex, semaphore, socket) ในหัวข้อ 68.2-68.4 แล้วสรุป
ด้วยหลักการออกแบบ (Rule of Zero/Three/Five ในหัวข้อ 68.5-68.6) และมาตรฐานความปลอดภัย
(Exception Safety Guarantee ในหัวข้อ 68.7) ที่ทำให้ระบบ RAII ทั้งหมดนี้แข็งแกร่งพอสำหรับใช้งาน
จริงระดับ production

**นี่คือเหตุผลที่แท้จริงที่ C++ ยังคงถูกเลือกใช้ในระบบที่ต้องการทั้งประสิทธิภาพสูงสุดและความ
ปลอดภัยของทรัพยากรสูงสุดพร้อมกัน** — ไม่ต้องพึ่ง garbage collector ที่ไม่อาจควบคุมเวลาการคืน
ทรัพยากรได้ และไม่ต้องพึ่งวินัยของโปรแกรมเมอร์ในการจำ `free`/`close`/`unlock` เอง เพราะ compiler
เป็นคนบังคับใช้กฎ RAII ให้ทุกครั้งโดยอัตโนมัติ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เขียน RAII wrapper แต่ลืมทำ Rule of Five ให้ครบ** — เขียนแค่ constructor/destructor แต่
   ลืม `= delete` copy หรือลืมเขียน move ทำให้ compiler generate copy constructor แบบ shallow
   copy ให้อัตโนมัติ ซึ่งจะนำไปสู่ double-close/double-free ทันทีที่มีคน copy object นั้น
2. **destructor, swap, หรือ move ที่ throw ได้** — ถ้า destructor throw ระหว่างที่ stack กำลัง
   unwind จาก exception อื่นอยู่แล้ว โปรแกรมจะเรียก `std::terminate()` ทันทีโดยไม่มีทางจับได้
   เลย ต้องทำให้ destructor/`swap()`/move เป็น `noexcept` เสมอ และห้ามมี code ที่ throw ได้อยู่
   ข้างในเด็ดขาด
3. **ใช้ `std::lock_guard` ในสถานการณ์ที่ต้องการปลดล็อกชั่วคราวหรือใช้กับ
   `condition_variable`** — `lock_guard` ไม่มี `unlock()`/`lock()` ให้เรียกเอง ต้องเปลี่ยนไปใช้
   `std::unique_lock` แทน
4. **เขียน class ที่ควรทำตาม Rule of Zero แต่ดันเขียน destructor เปล่าๆ ทิ้งไว้** (เช่น
   `~MyClass() {}`) — การมี destructor ที่ user-declared แม้จะว่างเปล่า **ปิดการสร้าง move
   constructor/assignment แบบอัตโนมัติของ compiler** (ตาม C++11 rules) ทำให้ class เสีย
   ประสิทธิภาพเพราะ compiler ถูกบังคับให้ใช้ copy แทน move ในหลายสถานการณ์ — ถ้าไม่จำเป็นต้อง
   เขียน destructor จริงๆ อย่าเขียนเลย ปล่อยให้ compiler generate ให้ (Rule of Zero)
5. **สับสนระหว่าง Basic กับ Strong Guarantee** — เข้าใจผิดว่าแค่ "ไม่ crash และไม่ leak" คือ
   strong guarantee แล้ว ทั้งที่จริงๆ นั่นคือ basic guarantee เท่านั้น strong guarantee ต้อง
   รับประกันว่า state เหมือนเดิมทุกประการเมื่อ throw
6. **ลืมว่า RAII wrapper ที่ห่อทรัพยากรของ OS (เช่น `pthread_mutex_t`, socket fd) ไม่ควร
   อนุญาตให้ copy ได้เลย** เพราะทรัพยากรเหล่านี้ผูกกับ OS โดยตรง การ copy ไม่มีความหมายอะไร
   (จะ copy "การเชื่อมต่อ socket เดียวกัน" ให้กลายเป็นสองอันได้อย่างไร) ให้ `= delete` เสมอ
   แล้วอนุญาตแค่ move ถ้าจำเป็น
7. **ไม่ได้ทดสอบ resource cleanup ในเส้นทางที่มี exception** — ทดสอบแค่ "happy path" ที่ไม่มี
   error เลย ทำให้พลาดบั๊กเรื่อง resource รั่วไหลที่เกิดเฉพาะตอน exception เท่านั้น ควรเขียน
   test case ที่บังคับให้เกิด error แล้วตรวจสอบด้วย valgrind เสมอ

---

## แบบฝึกหัดท้ายบท

1. เขียน RAII wrapper class ชื่อ `DynamicIntArray` ที่คุม `int*` ที่จองด้วย `new int[]` เอง
   ให้ครบตาม **Rule of Five** (constructor, destructor, copy constructor, copy assignment,
   move constructor, move assignment) แล้วเขียนโปรแกรมทดสอบที่สร้าง, copy, และ move object
   หลายตัว จากนั้นตรวจสอบด้วย `valgrind --leak-check=full` ว่าไม่มี memory leak เหลืออยู่เลย
2. เขียน RAII wrapper สำหรับ POSIX semaphore (`sem_t` จาก `<semaphore.h>`, ทบทวน Part 32) ชื่อ
   `PosixSemaphore` ที่เรียก `sem_init()` ใน constructor และ `sem_destroy()` ใน destructor
   พร้อมเขียน `SemaphoreGuard` แยกต่างหากที่เรียก `sem_wait()`/`sem_post()` แบบผูกกับ scope
   เหมือน `std::lock_guard`
3. Refactor class ต่อไปนี้ (ที่ละเมิด Rule of Three เพราะถือ raw pointer โดยไม่มี copy
   constructor) ให้กลายเป็น Rule of Zero โดยเปลี่ยน raw pointer เป็น RAII type ที่เหมาะสม:
   ```cpp
   class NameList {
   public:
       NameList(int capacity) : capacity_(capacity), names_(new std::string[capacity]) {}
       ~NameList() { delete[] names_; }
       // ขาด copy constructor/assignment -> ละเมิด Rule of Three!
   private:
       int capacity_;
       std::string* names_;
   };
   ```
4. เขียนฟังก์ชัน `transfer(Account& from, Account& to, int amount)` ที่โอนเงินระหว่างสอง
   `Account` (แต่ละตัวมี `std::mutex` ของตัวเอง) โดยใช้ `std::scoped_lock` ล็อกทั้งสอง mutex
   พร้อมกันแบบไม่เกิด deadlock แม้จะมีหลาย thread เรียก `transfer(a, b, ...)` และ
   `transfer(b, a, ...)` พร้อมกัน (ทบทวนปัญหา deadlock จาก lock ordering ที่ไม่ตรงกันใน
   Part 32.7-32.8)
5. จำแนกระดับ exception safety guarantee (basic/strong/nothrow) ของฟังก์ชันต่อไปนี้ พร้อมให้
   เหตุผลสั้นๆ: (ก) `std::vector::push_back` เมื่อ element เป็น type ที่มี move constructor
   แบบ `noexcept` (ข) ฟังก์ชันที่ลบไฟล์สองไฟล์ทีละไฟล์ โดยลบไฟล์แรกสำเร็จแต่ไฟล์ที่สอง
   `throw` เพราะไม่มีสิทธิ์เข้าถึง (ค) `std::swap` ของสอง `std::string`
6. เขียน RAII wrapper สำหรับ pipe สองทาง (`pipe()` จาก `<unistd.h>`, ทบทวน Part 29) ที่คืนค่า
   file descriptor สองตัว (read end กับ write end) โดย destructor ต้อง `close()` ทั้งสองด้าน
   ให้ครบถ้วน แม้จะมีด้านใดด้านหนึ่งถูกปิดไปก่อนแล้วด้วยมือก็ตาม (ป้องกัน double-close)

### แนวทางเฉลยข้อ 1

```cpp
// ex1_dynamic_array.cpp
#include <algorithm>
#include <iostream>
#include <utility>

class DynamicIntArray {
public:
    explicit DynamicIntArray(std::size_t size) : size_(size), data_(new int[size]{}) {
        std::cout << "  [จอง array ขนาด " << size_ << "]\n";
    }
    ~DynamicIntArray() {
        std::cout << "  [คืน array]\n";
        delete[] data_;
    }
    DynamicIntArray(const DynamicIntArray& other)
        : size_(other.size_), data_(new int[other.size_]) {
        std::copy(other.data_, other.data_ + size_, data_);
    }
    DynamicIntArray& operator=(const DynamicIntArray& other) {
        if (this != &other) {
            int* newData = new int[other.size_];
            std::copy(other.data_, other.data_ + other.size_, newData);
            delete[] data_;
            data_ = newData;
            size_ = other.size_;
        }
        return *this;
    }
    DynamicIntArray(DynamicIntArray&& other) noexcept
        : size_(std::exchange(other.size_, 0)), data_(std::exchange(other.data_, nullptr)) {}
    DynamicIntArray& operator=(DynamicIntArray&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            size_ = std::exchange(other.size_, 0);
            data_ = std::exchange(other.data_, nullptr);
        }
        return *this;
    }

    int& operator[](std::size_t i) { return data_[i]; }
    std::size_t size() const { return size_; }

private:
    std::size_t size_;
    int* data_;
};

int main() {
    DynamicIntArray a(5);
    for (std::size_t i = 0; i < a.size(); ++i) a[i] = static_cast<int>(i * 2);

    DynamicIntArray b = a;             // copy constructor
    DynamicIntArray c = std::move(a);  // move constructor
    b = c;                              // copy assignment

    std::cout << "b[3] = " << b[3] << ", c[3] = " << c[3] << '\n';
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 ex1_dynamic_array.cpp -o ex1
./ex1
valgrind --leak-check=full ./ex1
```

```
  [จอง array ขนาด 5]
b[3] = 6, c[3] = 6
  [คืน array]
  [คืน array]
  [คืน array]
```

(ผลลัพธ์จริงจะพิมพ์ `[คืน array]` สามครั้งตอนจบโปรแกรม ตามจำนวน object ที่ยังอยู่ในสแต็ก:
`a` หลัง move จะมี `data_ = nullptr` ทำให้ `delete[] nullptr` ปลอดภัยและไม่ทำอะไร, `b`, และ
`c`) valgrind จะรายงาน `All heap blocks were freed -- no leaks are possible` ยืนยันว่า Rule
of Five ที่เขียนครบถ้วนทำงานถูกต้องในทุกเส้นทาง (copy, move, และการทำลายตามปกติ)

### แนวทางเฉลยข้อ 3

เปลี่ยน raw pointer `std::string* names_` ให้เป็น `std::vector<std::string>` แทน — เมื่อ
สมาชิกทุกตัวเป็น RAII type แล้ว **ลบ destructor ทิ้งไปเลย** และปล่อยให้ compiler generate
ทุกอย่างให้ตาม Rule of Zero:

```cpp
// ex3_rule_of_zero.cpp
#include <iostream>
#include <string>
#include <vector>

class NameList {
public:
    explicit NameList(int capacity) : names_(static_cast<std::size_t>(capacity)) {}

    void set(std::size_t index, std::string name) { names_.at(index) = std::move(name); }
    const std::string& get(std::size_t index) const { return names_.at(index); }
    std::size_t size() const { return names_.size(); }

    // ไม่มี destructor, copy, หรือ move ที่เขียนเอง -> Rule of Zero
    // vector<string> จัดการทุกอย่างให้ถูกต้องอยู่แล้วภายในตัวมันเอง

private:
    std::vector<std::string> names_;
};

int main() {
    NameList list(3);
    list.set(0, "Somchai");
    list.set(1, "Somsri");
    list.set(2, "Suda");

    NameList copy = list;           // compiler-generated copy constructor ทำงานถูกต้อง (deep copy)
    NameList moved = std::move(list); // compiler-generated move constructor ทำงานถูกต้อง

    for (std::size_t i = 0; i < copy.size(); ++i) {
        std::cout << copy.get(i) << ' ';
    }
    std::cout << '\n';
    for (std::size_t i = 0; i < moved.size(); ++i) {
        std::cout << moved.get(i) << ' ';
    }
    std::cout << '\n';

    return 0;
}
```

```
Somchai Somsri Suda
Somchai Somsri Suda
```

การ refactor นี้ไม่ได้แค่ทำให้โค้ดสั้นลง — มันยัง**ลบบั๊กที่มีอยู่เดิมโดยอัตโนมัติ** เพราะ
`NameList` เวอร์ชันเดิมละเมิด Rule of Three (มี destructor ที่ custom แต่ไม่มี copy
constructor/assignment) ทำให้ถ้าใครเขียน `NameList copy = list;` กับเวอร์ชันเดิม จะเกิด
**shallow copy โดย compiler-generated copy constructor แบบ default** (เพราะไม่ได้ `= delete`
ไว้) ทำให้ทั้ง `list` และ `copy` ชี้ไปที่ `names_` ก้อนเดียวกัน แล้วทั้งคู่จะพยายาม
`delete[]` มันตอนหมด scope กลายเป็น **double free ทันที** — การเปลี่ยนไปใช้
`std::vector<std::string>` และทำตาม Rule of Zero แก้ปัญหานี้โดยไม่ต้องเขียน copy/move เอง
เลยแม้แต่บรรทัดเดียว

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- ทบทวนและขยายความ **RAII** อย่างจริงจังว่าเป็นหัวใจที่แท้จริงของ C++ ที่ทำให้การจัดการ
  ทรัพยากรปลอดภัยกว่าแนวทางของภาษาอื่น เพราะ compiler เป็นผู้บังคับใช้กฎ ไม่ใช่วินัยของ
  โปรแกรมเมอร์
- เห็นว่า RAII ไม่ได้จำกัดอยู่แค่ memory (ที่เรียนใน Part 67) แต่ครอบคลุมทรัพยากรทุกชนิดที่มี
  "เปิด" และ "ปิด": file handle, mutex lock (`std::lock_guard`/`std::unique_lock`), semaphore,
  และ socket
- เขียน **RAII wrapper class ของตัวเอง** สำหรับทรัพยากรที่ไม่มี smart pointer มาตรฐานรองรับ
  (`FILE*`, `pthread_mutex_t` จาก Module C, socket file descriptor) ตามพิมพ์เขียวมาตรฐานที่
  ใช้ได้กับทรัพยากรแทบทุกชนิด
- เข้าใจ **Rule of Zero** และเปรียบเทียบกับ **Rule of Three** (Part 46, 51) และ **Rule of
  Five** (จะเจาะลึกเต็มรูปแบบใน Part 70) ได้อย่างชัดเจนว่าแต่ละกฎใช้เมื่อไหร่
- จำแนกระดับ **Exception Safety Guarantee** ทั้งสามระดับ (basic, strong, nothrow) และรู้ว่า
  destructor/swap/move ต้องเป็น nothrow เสมอ
- รวบยอดทุกหลักการเป็น **RAII checklist ระดับ production** ที่ใช้ตรวจสอบโค้ดจริงได้ทันที

นี่คือการปิดท้าย **Module E — Templates, Generic Programming และ STL** อย่างสมบูรณ์ เราเดินทาง
จาก Template พื้นฐาน (Part 56-57) ผ่าน STL Container/Iterator/Algorithm ทั้งหมด (Part 58-65)
ไปจนถึงการจัดการหน่วยความจำและทรัพยากรระดับสูงสุด (Part 66-68) ซึ่งเป็นความรู้ที่จำเป็นสำหรับ
การเขียน C++ สมัยใหม่ระดับมืออาชีพทุกแขนง

ใน **Module F — Modern C++ (C++11 ถึง C++23)** ที่กำลังจะเริ่มต้นใน **Part 69** เราจะเจาะลึก
ฟีเจอร์ทั้งหมดที่ทำให้ C++ กลายเป็นภาษาที่ทันสมัยอย่างแท้จริง เริ่มจากภาพรวมของ **C++11** —
`auto`, `decltype`, range-based for, `nullptr`, และ `enum class` — ซึ่งหลายฟีเจอร์ในนั้นเราได้
ใช้งานจริงมาแล้วตลอดหลักสูตรนี้โดยไม่รู้ตัว (เช่น `auto` ใน range-based for loop ของ
`std::vector`) ถึงเวลาที่จะเข้าใจที่มาและกลไกเบื้องหลังของมันอย่างเป็นทางการ

**ต่อไป:** [Part 69 — ภาพรวม C++11](./part-069-cpp11-overview.md)
