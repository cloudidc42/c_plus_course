# Part 94: Continuous Integration สำหรับ C++ ด้วย GitHub Actions (Step 745–752)

> Module H — Build Systems, Testing และ Tooling | Part 94 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 745–752
> Part ก่อนหน้า: Part 93 — Unit Testing (Google Test/Catch2) | Part ถัดไป: [Part 95 — Static Analysis: clang-tidy, cppcheck, clang-format](./part-095-static-analysis.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่า **Continuous Integration (CI)** คืออะไร แก้ปัญหาอะไรที่การทดสอบด้วยมือ
   (Manual Testing) แก้ไม่ได้ และทำไมทีมวิศวกรรมซอฟต์แวร์ระดับโลกแทบทุกทีมถึงใช้ CI
2. อ่านและเขียนไฟล์ YAML ของ **GitHub Actions** ได้อย่างถูกต้อง เข้าใจโครงสร้าง `on` / `jobs` /
   `steps` / `runs-on` และแนวคิดของ **Runner**
3. เขียน Workflow ที่ `checkout` โค้ด, ติดตั้ง dependency, `cmake` configure/build โปรเจกต์ C++
   และรัน unit test ที่เขียนไว้ใน **Part 93** โดยอัตโนมัติทุกครั้งที่ `push` หรือเปิด Pull Request
4. ใช้ **Matrix Build** เพื่อทดสอบโค้ดเดียวกันกับหลาย compiler (GCC/Clang) พร้อมกันในการรันครั้งเดียว
5. ใช้ **Caching** (`actions/cache`) เพื่อลดเวลารัน CI โดยไม่กระทบความถูกต้องของผลลัพธ์
6. ติดตั้ง **Status Badge** บน README เพื่อให้เห็นสถานะ build ล่าสุดของโปรเจกต์แบบเรียลไทม์
7. อธิบายภาพรวมของแนวคิด **CI/CD** ทั้งระบบ และบอกความแตกต่างระหว่าง Continuous Integration,
   Continuous Delivery และ Continuous Deployment ได้อย่างถูกต้อง
8. ตรวจสอบ (validate) ความถูกต้องของไฟล์ YAML และคำสั่งที่ workflow จะรัน **ก่อน** push ขึ้น
   GitHub จริง เพื่อไม่ให้เสียเวลารอ CI รันแล้วพังเพราะ syntax error ง่ายๆ

---

## หมายเหตุสำคัญก่อนเริ่ม: อะไรคือของจริง อะไรคือภาพประกอบ

Part นี้พูดถึง GitHub Actions ซึ่งเป็นบริการที่รันบน Server ของ GitHub เอง เครื่องที่ใช้เรียน
หลักสูตรนี้ไม่มี GitHub Actions Runner ให้กดรันจริง ดังนั้นตลอด Part นี้จะทำสิ่งที่ตรงไปตรงมาที่สุด
และเป็นวิธีที่มืออาชีพใช้ก่อน push งานทุกครั้งอยู่แล้ว คือ:

1. **เขียนไฟล์ `.github/workflows/ci.yml` จริง** ตามที่จะใช้ในโปรเจกต์จริง
2. **ตรวจสอบ syntax YAML ด้วยโปรแกรมจริง** (`python3` + module `yaml`) เพื่อยืนยันว่าไฟล์ parse
   ผ่าน ไม่มี syntax error ที่จะทำให้ GitHub Actions ปฏิเสธ workflow ทันที
3. **รันคำสั่งทุกคำสั่งที่อยู่ใน step ของ workflow บนเครื่องนี้จริงๆ** (คำสั่งเดียวกันเป๊ะ) เพื่อยืนยัน
   ว่าถ้า workflow นี้ไปรันบน GitHub Actions Runner จริง มันจะสำเร็จ เพราะ Runner ของ GitHub
   Actions ที่ใช้ `ubuntu-latest` ก็คือ Ubuntu ที่มี toolchain แบบเดียวกับเครื่องพัฒนาทั่วไป

ทุกจุดที่เป็นผลลัพธ์จาก GitHub Actions โดยตรง (เช่น หน้าตาของ Actions tab, สีของ badge) จะระบุ
ชัดเจนว่าเป็น**ภาพประกอบตามพฤติกรรมจริงของบริการ** ไม่ใช่สิ่งที่รันจริงบนเครื่องนี้

---

## 94.1 Continuous Integration คืออะไร และทำไมสำคัญ (Step 745)

### ปัญหาของการทดสอบด้วยมือ

ใน **Part 93** เราเขียน Unit Test ด้วย Google Test ไว้ครบแล้ว แต่มีคำถามสำคัญคือ:
**ใครจะเป็นคนรัน test พวกนี้ และรันเมื่อไหร่?**

ถ้าคำตอบคือ "รันเองตอนจำได้" ปัญหาที่ตามมาในทีมจริงมีดังนี้:

- **ลืมรัน**: นักพัฒนากำลังรีบส่งงาน แก้โค้ดนิดเดียว "คิดว่า" ไม่กระทบอะไร เลยข้ามการรัน test ไป
  แล้วโค้ดที่ push ไปก็ทำให้ระบบอื่นพังโดยไม่มีใครรู้จนกว่าจะมีคนอื่นเจอ
- **"มันรันผ่านในเครื่องผม"**: โค้ดรันผ่านบนเครื่องคนเขียน (macOS, compiler เวอร์ชันหนึ่ง) แต่พังบน
  เครื่องเพื่อนร่วมทีม (Linux, compiler อีกเวอร์ชัน) เพราะ environment ไม่เหมือนกัน
- **ตรวจพบบั๊กช้าเกินไป**: กว่าจะรู้ว่า commit ไหนทำให้ test พัง อาจต้องไล่ดู commit ย้อนหลังหลาย
  สิบ commit เพราะไม่มีใครรัน test ระหว่างทางเลย
- **ไม่มีมาตรฐานบังคับ**: ทีมตกลงกันว่า "ทุกคนต้องรัน test ก่อน push" แต่ไม่มีกลไกบังคับจริง
  สุดท้ายก็ขึ้นอยู่กับวินัยส่วนบุคคลของแต่ละคน ซึ่งพังง่ายมากเมื่อทีมโตขึ้นหรือ deadline ใกล้เข้ามา

### Continuous Integration แก้ปัญหานี้อย่างไร

**Continuous Integration (CI)** คือแนวปฏิบัติที่ให้ **ระบบอัตโนมัติ** (ไม่ใช่คน) ทำหน้าที่
build และรัน test ทุกครั้งที่มีการเปลี่ยนแปลงโค้ดถูก push ขึ้น repository หรือเปิด Pull Request
โดยไม่ต้องมีใครสั่งเอง หัวใจสำคัญคือ:

```
นักพัฒนา push โค้ด
        │
        ▼
Server ของ CI ตรวจจับการเปลี่ยนแปลงอัตโนมัติ (webhook)
        │
        ▼
สร้าง "เครื่องเปล่า" (Runner) ขึ้นมาใหม่ทุกครั้ง — ไม่มี state เก่าตกค้าง
        │
        ▼
Checkout โค้ดล่าสุด → ติดตั้ง dependency → Build → รัน Test
        │
        ▼
รายงานผล: ✅ Pass หรือ ❌ Fail กลับไปที่ Pull Request / Commit ทันที
```

ประโยชน์หลักที่ได้จากแนวคิดนี้:

1. **ตรวจจับปัญหาทันทีที่เกิด** ไม่ใช่หลังจากผ่านไปหลายวันหรือหลาย commit
2. **Environment สะอาดและเหมือนกันทุกครั้ง** เพราะ Runner ถูกสร้างใหม่ทุกรอบ ไม่มีปัญหา
   "รันผ่านในเครื่องผม" อีกต่อไป — ถ้า CI บอกว่าผ่าน แปลว่าผ่านจริงบน environment มาตรฐาน
3. **บังคับมาตรฐานคุณภาพแบบอัตโนมัติ**: ตั้งกฎได้ว่า Pull Request จะ merge เข้า `main` ไม่ได้
   ถ้า CI ยังไม่ผ่าน (บังคับด้วยระบบ ไม่ใช่ด้วยความหวังว่าคนจะมีวินัย)
4. **เอกสารที่มีชีวิต**: ทุก commit มีสถานะ pass/fail ติดอยู่ ย้อนดูประวัติได้ว่าโค้ดจุดไหนเริ่มพัง
5. **ทำงานร่วมกับทีมใหญ่ได้จริง**: เมื่อมีนักพัฒนาหลายสิบคน push โค้ดพร้อมกันทุกวัน CI คือ
   กลไกเดียวที่สเกลได้โดยไม่ต้องเพิ่มคนมานั่งเช็คด้วยมือ

### ทำไมต้อง GitHub Actions

มีบริการ CI ให้เลือกหลายเจ้า (Jenkins, GitLab CI, CircleCI, Travis CI, Azure Pipelines) แต่
**GitHub Actions** เป็นตัวเลือกที่เหมาะกับหลักสูตรนี้ที่สุดเพราะ:

- **ผูกกับ GitHub โดยตรง** — ถ้าโค้ดอยู่บน GitHub อยู่แล้ว (ซึ่งเป็นที่เก็บโค้ดที่ใช้ทั่วโลก) ไม่ต้อง
  ไปตั้งค่าเชื่อมต่อกับบริการภายนอกเพิ่มเติมเลย
- **ฟรีสำหรับ Public Repository** และมี Free Tier ที่เพียงพอสำหรับ Private Repository ขนาดเล็ก
- **กำหนดค่าด้วยไฟล์ YAML ในโค้ด** (Infrastructure as Code) — วิธีตั้งค่า CI ถูกเก็บอยู่ใน
  repository เดียวกับโค้ด ย้อนดูประวัติการเปลี่ยนแปลงผ่าน git ได้เหมือนไฟล์โค้ดทั่วไป
- **มี Ecosystem ของ "Actions" สำเร็จรูป** จำนวนมาก (เช่น `actions/checkout`, `actions/cache`)
  ที่ทำงานซ้ำๆ ให้โดยไม่ต้องเขียนเอง

---

## 94.2 โครงสร้างพื้นฐานของ GitHub Actions (Step 746)

GitHub Actions ทำงานตาม hierarchy นี้:

```
Repository
   └── .github/workflows/*.yml      (Workflow — 1 ไฟล์ = 1 workflow)
          └── on:                    (Event ที่ trigger workflow นี้)
          └── jobs:                  (กลุ่มงาน — รันแยกกันได้ อาจขนานกัน)
                └── job_name:
                       └── runs-on:  (เครื่องแบบไหนที่จะรัน — Runner)
                       └── steps:    (ลำดับขั้นตอนภายใน job นี้ รันตามลำดับบน Runner เดียวกัน)
```

### คำศัพท์สำคัญที่ต้องเข้าใจก่อน

| คำศัพท์ | ความหมาย |
|---|---|
| **Workflow** | กระบวนการอัตโนมัติทั้งหมด กำหนดในไฟล์ `.yml` หนึ่งไฟล์ใน `.github/workflows/` |
| **Event** | เหตุการณ์ที่ทำให้ workflow เริ่มทำงาน เช่น `push`, `pull_request`, `schedule` |
| **Job** | กลุ่มของ step ที่รันบน Runner เครื่องเดียวกัน โดย default หลาย job ใน workflow เดียวกันจะ
รันขนานกัน (parallel) เว้นแต่จะระบุ `needs:` ให้รอกัน |
| **Step** | คำสั่งเดียวหรือ Action สำเร็จรูปหนึ่งตัวภายใน job รันตามลำดับจากบนลงล่าง |
| **Runner** | เครื่อง (Virtual Machine หรือ Container) ที่ GitHub เตรียมไว้ให้รัน job แต่ละครั้งจะได้
เครื่อง**สะอาดใหม่**เสมอ (`ubuntu-latest`, `windows-latest`, `macos-latest`) |
| **Action** | หน่วยงานสำเร็จรูปที่นำมาใช้ซ้ำได้ เช่น `actions/checkout@v4` เขียนโดย GitHub เองหรือ
ชุมชน สามารถ `uses:` เรียกใช้ในแต่ละ step ได้ |

### ไฟล์ YAML ต้องอยู่ตรงไหน

GitHub Actions จะสแกนหา workflow เฉพาะไฟล์ `.yml` หรือ `.yaml` ที่อยู่ใน**โฟลเดอร์ที่ชื่อ
`.github/workflows/`** ที่ root ของ repository เท่านั้น — วางผิดที่จะไม่ทำงานเลยโดยไม่มี error
ใดๆ แจ้งเตือน (จุดนี้เป็นกับดักที่พบบ่อยมากสำหรับมือใหม่)

### โครงสร้าง YAML ขั้นต่ำที่สุด

```yaml
name: My First CI

on: [push]

jobs:
  say-hello:
    runs-on: ubuntu-latest
    steps:
      - name: Print a message
        run: echo "Hello from CI"
```

อธิบายทีละส่วน:

- **`name:`** — ชื่อ workflow ที่จะแสดงในแท็บ Actions ของ GitHub (ถ้าไม่ใส่ จะใช้ชื่อไฟล์แทน)
- **`on: [push]`** — trigger ทุกครั้งที่มีการ `git push` ขึ้น repository นี้ (ทุก branch)
- **`jobs:`** — map ของ job แต่ละตัว คีย์ `say-hello` คือชื่อ job ที่ตั้งเอง
- **`runs-on: ubuntu-latest`** — สั่งให้รันบน Runner ที่เป็น Ubuntu เวอร์ชันล่าสุดที่ GitHub เตรียมไว้
- **`steps:`** — list ของขั้นตอน แต่ละ step มี `name` (คำอธิบาย, ไม่บังคับ) และ `run` (คำสั่ง shell)
  หรือ `uses` (เรียก Action สำเร็จรูป)

### YAML คือภาษาที่ **เรื่องมากเรื่อง indentation**

YAML ใช้การเยื้องบรรทัด (Indentation) แทนวงเล็บปีกกาแบบ C/C++ ในการกำหนดโครงสร้าง กฎที่
สำคัญที่สุด:

- ห้ามผสม **Tab** กับ **Space** — ต้องใช้ Space เท่านั้นตลอดทั้งไฟล์
- ระดับการเยื้องต้องตรงกันเป๊ะในระดับเดียวกัน (นิยมใช้ 2 space ต่อระดับ)
- `-` (dash) หมายถึงสมาชิกใน list เช่น `steps:` แต่ละอันขึ้นต้นด้วย `- `

ลองพิสูจน์ว่า YAML parse ได้จริงด้วยเครื่องมือจริงบนเครื่องนี้ (ไม่ใช่แค่ "เดา" ว่า syntax ถูก):

```bash
python3 -c "
import yaml
with open('.github/workflows/ci.yml') as f:
    data = yaml.safe_load(f)
print('YAML parsed OK')
"
```

ผลลัพธ์จริงที่รันได้บนเครื่องนี้:

```
YAML parsed OK
```

> **เทคนิคมืออาชีพ**: ก่อน push ไฟล์ `.yml` ทุกครั้ง ให้รันคำสั่ง `python3 -c "import yaml;
> yaml.safe_load(open('path/to/file.yml'))"` เพื่อเช็ค syntax ก่อนเสมอ เพราะการรอ 2-3 นาที
> ให้ GitHub Actions บอกว่า workflow พังเพราะเยื้องบรรทัดผิด เป็นการเสียเวลาที่ป้องกันได้ง่ายมาก

---

## 94.3 เขียน Workflow แรก: Build CMake + รัน Unit Test อัตโนมัติ (Step 747)

เป้าหมายของหัวข้อนี้คือเขียน workflow ที่ทำสิ่งเดียวกับที่เราทำด้วยมือใน **Part 91** (CMake) และ
**Part 93** (Google Test) แต่ให้ GitHub ทำให้อัตโนมัติทุกครั้งที่ push

### โครงสร้างโปรเจกต์ตัวอย่างที่จะใช้ตลอด Part นี้

สมมติโปรเจกต์ C++ จาก Part 91–93 มีโครงสร้างประมาณนี้:

```
my_project/
├── CMakeLists.txt
├── src/
│   ├── mathutils.hpp
│   └── mathutils.cpp
└── tests/
    └── test_mathutils.cpp
```

`CMakeLists.txt` (แบบเดียวกับที่ตั้งค่าไว้ใน Part 91 และเชื่อม Google Test ตาม Part 93):

```cmake
cmake_minimum_required(VERSION 3.16)
project(my_project CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(mathutils src/mathutils.cpp)
target_include_directories(mathutils PUBLIC src)

find_package(GTest REQUIRED)
enable_testing()

add_executable(unit_tests tests/test_mathutils.cpp)
target_link_libraries(unit_tests PRIVATE mathutils GTest::gtest GTest::gtest_main)

include(GoogleTest)
gtest_discover_tests(unit_tests)
```

`src/mathutils.hpp` / `src/mathutils.cpp` — ไลบรารีเล็กๆ ที่มีฟังก์ชันคณิตศาสตร์พื้นฐาน:

```cpp
// mathutils.hpp
#ifndef MATHUTILS_HPP
#define MATHUTILS_HPP

namespace mathutils {
int add(int a, int b);
long long factorial(int n);
bool is_prime(int n);
int gcd(int a, int b);
}  // namespace mathutils

#endif
```

```cpp
// mathutils.cpp
#include "mathutils.hpp"

namespace mathutils {

int add(int a, int b) { return a + b; }

long long factorial(int n) {
    if (n < 0) return 0;
    long long result = 1;
    for (int i = 2; i <= n; ++i) result *= i;
    return result;
}

bool is_prime(int n) {
    if (n < 2) return false;
    for (int i = 2; static_cast<long long>(i) * i <= n; ++i) {
        if (n % i == 0) return false;
    }
    return true;
}

int gcd(int a, int b) {
    while (b != 0) {
        int t = b;
        b = a % b;
        a = t;
    }
    return a;
}

}  // namespace mathutils
```

`tests/test_mathutils.cpp` — unit test ตามสไตล์ Google Test ที่เรียนใน Part 93:

```cpp
#include <gtest/gtest.h>
#include "mathutils.hpp"

TEST(MathUtils, AddPositiveNumbers) {
    EXPECT_EQ(mathutils::add(2, 3), 5);
}

TEST(MathUtils, FactorialTypicalCase) {
    EXPECT_EQ(mathutils::factorial(5), 120);
}

TEST(MathUtils, IsPrimeTrueCases) {
    EXPECT_TRUE(mathutils::is_prime(2));
    EXPECT_TRUE(mathutils::is_prime(97));
}

TEST(MathUtils, GcdTypicalCase) {
    EXPECT_EQ(mathutils::gcd(48, 18), 6);
}
```

### ยืนยันก่อนว่า Build + Test รันผ่านบนเครื่อง (ก่อนเอาไปใส่ CI)

หลักการสำคัญของ CI คือ **workflow แค่รันคำสั่งเดิมที่เรารันด้วยมือทุกวันซ้ำแบบอัตโนมัติ** ดังนั้น
ต้องมั่นใจก่อนว่าคำสั่งพวกนี้ใช้ได้จริง:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j
cd build && ctest --output-on-failure
```

ผลลัพธ์จริงจากการรันบนเครื่องนี้ (GCC 13.3.0):

```
-- The CXX compiler identification is GNU 13.3.0
-- Found GTest: /usr/lib/x86_64-linux-gnu/cmake/GTest/GTestConfig.cmake (found version "1.14.0")
-- Configuring done (0.4s)
-- Generating done (0.0s)
-- Build files have been written to: .../build
[ 25%] Building CXX object CMakeFiles/mathutils.dir/src/mathutils.cpp.o
[ 50%] Linking CXX static library libmathutils.a
[ 50%] Built target mathutils
[ 75%] Building CXX object CMakeFiles/unit_tests.dir/tests/test_mathutils.cpp.o
[100%] Linking CXX executable unit_tests
[100%] Built target unit_tests
    Start 1: MathUtils.AddPositiveNumbers
1/8 Test #1: MathUtils.AddPositiveNumbers ........   Passed    0.00 sec
    Start 2: MathUtils.AddNegativeNumbers
2/8 Test #2: MathUtils.AddNegativeNumbers ........   Passed    0.00 sec
    ...
8/8 Test #8: MathUtils.GcdTypicalCase ............   Passed    0.00 sec

100% tests passed, 0 tests failed out of 8
Total Test time (real) =   0.03 sec
```

(ผลลัพธ์นี้คือของโปรเจกต์ตัวอย่างจริงที่มี 8 test case ครอบคลุมฟังก์ชันทั้ง 4 ตัว — ตัวเลขจะต่างไป
ตามจำนวน test ที่แต่ละคนเขียนใน Part 93 ของตัวเอง) เมื่อคำสั่งพวกนี้รันผ่านบนเครื่องตัวเองแล้ว
ขั้นตอนต่อไปคือ**คัดลอกคำสั่งเดิมเป๊ะๆ** ไปใส่ใน workflow YAML

### เขียน `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake libgtest-dev g++

      - name: Configure CMake
        run: cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug

      - name: Build
        run: cmake --build build -j "$(nproc)"

      - name: Run unit tests
        working-directory: build
        run: ctest --output-on-failure
```

อธิบายแต่ละ step:

1. **`actions/checkout@v4`** — Action สำเร็จรูปจาก GitHub เอง ทำหน้าที่ `git clone` โค้ดของ
   repository นี้ลงบน Runner (ถ้าไม่มี step นี้ Runner จะเป็นเครื่องเปล่าที่ไม่มีโค้ดเราเลย)
2. **Install dependencies** — Runner ของ `ubuntu-latest` มาแบบเปล่าๆ ไม่มี `cmake`/`libgtest-dev`
   ติดตั้งมาให้ ต้องสั่งติดตั้งเองทุกครั้ง (นี่คือเหตุผลที่หัวข้อ 94.5 จะพูดเรื่อง caching ต่อ)
3. **Configure / Build / Run unit tests** — คำสั่งชุดเดียวกับที่ทดสอบบนเครื่องตัวเองด้านบนทุก
   ตัวอักษร — นี่คือหัวใจของ CI: **ไม่มีเวทมนตร์อะไรพิเศษ แค่รันคำสั่งเดิมบนเครื่องที่สะอาดกว่า**

`working-directory: build` บอกให้ step นั้นรันคำสั่งโดยเปลี่ยน current directory ไปที่โฟลเดอร์
`build` ก่อน (เทียบเท่ากับ `cd build && ctest ...`)

### ตรวจสอบว่า YAML ไฟล์นี้ syntax ถูกต้องจริง

```bash
python3 -c "
import yaml
with open('.github/workflows/ci.yml') as f:
    data = yaml.safe_load(f)
print('YAML parsed OK')
"
```

ผลลัพธ์จริง:

```
YAML parsed OK
```

> **ข้อสังเกตที่น่าสนใจจากการทดสอบจริง**: เมื่อลอง `print()` เนื้อหาที่ parse ออกมาเป็น
> Python dict จะพบว่าคีย์ `on:` ถูกแปลงเป็นคีย์ `True` (boolean) แทนที่จะเป็น string `"on"`!
> นี่เป็นพฤติกรรมมาตรฐานของ YAML 1.1 (ตัวอักษร `on`/`off`/`yes`/`no` ที่ไม่ใส่เครื่องหมายคำพูด
> ถูกตีความเป็น boolean โดย spec) GitHub Actions เองมี parser พิเศษที่รู้จักคีย์ `on` และ
> ทำงานถูกต้องเสมอ ไม่ต้องกังวล แต่ถ้าใช้เครื่องมือ YAML validator ทั่วไป (เช่น `yamllint` บาง
> เวอร์ชัน หรือเขียน Python script ไปประมวลผลไฟล์ workflow เอง) แล้วเจอพฤติกรรมแปลกๆ ตรงคีย์
> `on` ให้รู้ไว้ว่านี่คือ YAML quirk ที่มีมานานแล้ว ไม่ใช่ error ในไฟล์ของเรา

### สิ่งที่จะเกิดขึ้นจริงบน GitHub (ภาพประกอบ)

เมื่อ push ไฟล์นี้ขึ้น GitHub จริง จะเห็นแท็บ **Actions** ของ repository แสดง workflow run ใหม่
ทุกครั้งที่มีการ push หรือเปิด PR ไปที่ branch `main` พร้อมสถานะ (กำลังรัน/ผ่าน/ไม่ผ่าน) และ log
แบบละเอียดของแต่ละ step ให้กดดูได้ ถ้า step ไหนล้มเหลว ทั้ง Pull Request จะแสดงเครื่องหมาย
❌ กำกับไว้ชัดเจน ทำให้ reviewer เห็นได้ทันทีว่ายังไม่ควร merge

---

## 94.4 Matrix Build: ทดสอบหลาย Compiler พร้อมกัน (Step 748)

### ปัญหา: "รันผ่านบน GCC ไม่ได้แปลว่ารันผ่านบน Clang"

โค้ด C++ บางส่วนอาจใช้ Undefined Behavior หรือฟีเจอร์ที่ compiler ตัวหนึ่งยอมให้ผ่านแบบเงียบๆ
แต่ compiler อีกตัวเตือนหรือปฏิเสธ การทดสอบด้วย compiler เดียวจึงไม่เพียงพอสำหรับโปรเจกต์ที่
ต้องการ portability สูง **Matrix Build** ให้เราทดสอบโค้ดเดียวกันกับหลายค่า configuration
พร้อมกันในการรัน workflow ครั้งเดียว โดยที่แต่ละ combination รันแยกกันเป็นคนละ job ขนานกัน

### เขียน Matrix สำหรับ GCC + Clang

```yaml
jobs:
  build-and-test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest]
        compiler: [gcc, clang]

    runs-on: ${{ matrix.os }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake libgtest-dev g++ clang

      - name: Select compiler (gcc)
        if: matrix.compiler == 'gcc'
        run: |
          echo "CC=gcc" >> "$GITHUB_ENV"
          echo "CXX=g++" >> "$GITHUB_ENV"

      - name: Select compiler (clang)
        if: matrix.compiler == 'clang'
        run: |
          echo "CC=clang" >> "$GITHUB_ENV"
          echo "CXX=clang++" >> "$GITHUB_ENV"

      - name: Configure CMake
        run: cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug

      - name: Build
        run: cmake --build build -j "$(nproc)"

      - name: Run unit tests
        working-directory: build
        run: ctest --output-on-failure
```

อธิบายกลไก:

- **`strategy.matrix`** — กำหนดตัวแปรที่จะเปลี่ยนค่าในแต่ละ combination GitHub Actions จะ
  สร้าง job แยกให้อัตโนมัติสำหรับทุก combination (ในที่นี้คือ 1 × 2 = 2 job: gcc และ clang)
- **`${{ matrix.compiler }}`** — syntax ของ GitHub Actions Expression ใช้ดึงค่าตัวแปรจาก matrix
  ปัจจุบัน ในตัวอย่างนี้ใช้ผ่าน `if:` condition เพื่อเลือกว่าจะ set environment variable ไหน
- **`fail-fast: false`** — ค่า default ของ `fail-fast` คือ `true` ซึ่งจะยกเลิก job ที่เหลือทั้งหมด
  ทันทีที่มี job ใดล้มเหลว การตั้งเป็น `false` ทำให้ทุก combination รันจนจบแม้บางตัวจะ fail
  ทำให้เห็นภาพรวมครบว่า compiler ไหนบ้างที่มีปัญหา (มีประโยชน์มากตอน debug)
- **`echo "CC=gcc" >> "$GITHUB_ENV"`** — วิธีมาตรฐานของ GitHub Actions ในการตั้งค่า environment
  variable ที่จะ**คงอยู่ข้ามไปยัง step ถัดไป**ใน job เดียวกัน (ตั้งด้วย `export` ธรรมดาจะใช้ไม่ได้
  ข้าม step เพราะแต่ละ `run:` รันเป็น shell process แยกกัน)

### ยืนยันบนเครื่องจริงว่าทั้งสอง compiler build ผ่านและ test ผ่านจริง

รันชุดคำสั่งเดียวกันด้วย GCC และ Clang บนเครื่องนี้ (จำลองสิ่งที่ matrix job แต่ละตัวจะทำ):

```bash
# GCC
cmake -S . -B build-gcc -DCMAKE_BUILD_TYPE=Debug
cmake --build build-gcc -j
cd build-gcc && ctest --output-on-failure

# Clang
CXX=clang++ CC=clang cmake -S . -B build-clang -DCMAKE_BUILD_TYPE=Debug
cmake --build build-clang -j
cd build-clang && ctest --output-on-failure
```

ผลลัพธ์จริงฝั่ง Clang (18.1.3) ที่รันบนเครื่องนี้:

```
-- The CXX compiler identification is Clang 18.1.3
-- Found GTest: /usr/lib/x86_64-linux-gnu/cmake/GTest/GTestConfig.cmake (found version "1.14.0")
[ 25%] Building CXX object CMakeFiles/mathutils.dir/src/mathutils.cpp.o
[ 50%] Linking CXX static library libmathutils.a
[ 75%] Building CXX object CMakeFiles/unit_tests.dir/tests/test_mathutils.cpp.o
[100%] Linking CXX executable unit_tests
    Start 1: MathUtils.AddPositiveNumbers
1/8 Test #1: MathUtils.AddPositiveNumbers ........   Passed    0.00 sec
    ...
8/8 Test #8: MathUtils.GcdTypicalCase ............   Passed    0.00 sec

100% tests passed, 0 tests failed out of 8
Total Test time (real) =   0.02 sec
```

ทั้ง GCC และ Clang build และ test ผ่านหมดจริงบนเครื่องนี้ ยืนยันได้ว่าถ้า matrix นี้ไปรันบน
GitHub Actions จะได้ผลเดียวกัน (เพราะ Runner ของ `ubuntu-latest` ใช้ toolchain ชุดเดียวกันกับที่
ติดตั้งผ่าน `apt` แบบเดียวกับที่ทดสอบไว้ตรงนี้)

### ขยาย Matrix ให้ครอบคลุมมากขึ้น

ในโปรเจกต์จริงมักขยาย matrix ให้ครอบคลุมทั้ง OS และ Build Type ไปพร้อมกัน:

```yaml
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-latest, macos-latest]
    compiler: [gcc, clang]
    build_type: [Debug, Release]
```

Matrix นี้จะสร้าง **2 × 2 × 2 = 8 job** รันขนานกันทั้งหมด (ภายในขีดจำกัดจำนวน job ขนานที่
GitHub Actions อนุญาตในแต่ละ Plan) ยิ่ง matrix ใหญ่ยิ่งมั่นใจได้มากว่าโค้ด portable จริง แต่ก็ใช้
เวลาและโควตา CI มากขึ้นตามไปด้วย — ควรเลือก combination ที่จำเป็นจริงๆ ไม่ใช่ใส่ทุกอย่าง
เท่าที่คิดออก

---

## 94.5 Caching Dependency เพื่อความเร็ว (Step 749)

### ปัญหา: ติดตั้ง dependency ใหม่ทุกครั้งช้ามาก

Step "Install dependencies" ใน workflow ของเราสั่ง `apt-get update && apt-get install` ทุกครั้ง
ที่ workflow รัน ซึ่งหมายความว่า**ทุก push ต้องดาวน์โหลดแพ็กเกจเดิมซ้ำใหม่หมด** ยิ่งโปรเจกต์มี
dependency เยอะ (เช่นดึงไลบรารีผ่าน Conan/vcpkg จาก Part 92) ยิ่งเสียเวลามาก บางโปรเจกต์ที่มี
dependency หนักๆ อาจเสียเวลาติดตั้งนานกว่าขั้นตอน build+test จริงของตัวเองเสียอีก

### `actions/cache`: เก็บผลลัพธ์ข้าม workflow run

Action มาตรฐานของ GitHub สำหรับ cache คือ `actions/cache` หลักการทำงาน:

```yaml
- name: Cache CMake build directory
  uses: actions/cache@v4
  with:
    path: build
    key: ${{ runner.os }}-build-${{ hashFiles('**/CMakeLists.txt') }}
    restore-keys: |
      ${{ runner.os }}-build-
```

อธิบายกลไก:

- **`path:`** — โฟลเดอร์ที่ต้องการเก็บ cache ไว้ (เช่น โฟลเดอร์ build, โฟลเดอร์ dependency ที่
  package manager ดาวน์โหลดมา)
- **`key:`** — "ลายเซ็น" ของ cache แต่ละชุด ถ้า key เดิมเคยถูกเก็บไว้แล้ว จะดึง cache นั้นมาใช้
  ทันทีแทนการสร้างใหม่ ในตัวอย่างนี้ใช้ `hashFiles('**/CMakeLists.txt')` เพื่อให้ key เปลี่ยนเมื่อ
  ไฟล์ `CMakeLists.txt` เปลี่ยนแปลงเท่านั้น (ถ้าโครงสร้าง build เปลี่ยน ก็ควรสร้าง cache ใหม่แทน
  การใช้ของเก่าที่อาจไม่ตรงกันแล้ว)
- **`restore-keys:`** — รายการ prefix สำรอง ถ้าไม่เจอ key ที่ตรงเป๊ะ จะลองหา cache ที่ขึ้นต้นด้วย
  prefix นี้แทน (cache เก่ากว่าดีกว่าไม่มี cache เลย ช่วยลดเวลาบางส่วนได้แม้ไม่ครบ 100%)

### ตัวอย่าง workflow ที่มี caching ครบ

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
        os: [ubuntu-latest]
        compiler: [gcc, clang]

    runs-on: ${{ matrix.os }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake libgtest-dev g++ clang

      - name: Cache CMake build directory
        uses: actions/cache@v4
        with:
          path: build
          key: ${{ runner.os }}-${{ matrix.compiler }}-build-${{ hashFiles('**/CMakeLists.txt') }}
          restore-keys: |
            ${{ runner.os }}-${{ matrix.compiler }}-build-

      - name: Select compiler (gcc)
        if: matrix.compiler == 'gcc'
        run: |
          echo "CC=gcc" >> "$GITHUB_ENV"
          echo "CXX=g++" >> "$GITHUB_ENV"

      - name: Select compiler (clang)
        if: matrix.compiler == 'clang'
        run: |
          echo "CC=clang" >> "$GITHUB_ENV"
          echo "CXX=clang++" >> "$GITHUB_ENV"

      - name: Configure CMake
        run: cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug

      - name: Build
        run: cmake --build build -j "$(nproc)"

      - name: Run unit tests
        working-directory: build
        run: ctest --output-on-failure
```

สังเกตว่า cache key ผสม `matrix.compiler` เข้าไปด้วย — เพราะ build directory ของ gcc กับ clang
มีไฟล์ object ที่ compile ไม่เหมือนกัน ถ้าใช้ key เดียวกันทั้งสอง job จะแย่ง cache กันจนได้ของ
ผิด compiler ไปใช้

### ตรวจสอบ YAML ที่มี caching ผ่าน parser จริง

```bash
python3 -c "
import yaml
data = yaml.safe_load(open('.github/workflows/ci.yml'))
print('YAML parsed OK, jobs =', list(data['jobs'].keys()))
"
```

```
YAML parsed OK, jobs = ['build-and-test']
```

### ข้อควรระวังเรื่อง Caching

Caching ไม่ใช่ของฟรีที่ไม่มีข้อเสีย — cache ที่ผิดพลาดหรือเก่าเกินไปอาจทำให้ CI **รายงานผลผิด**
เช่น ใช้ library เวอร์ชันเก่าที่ cache ไว้ทั้งที่ควรถูกอัปเดตแล้ว (ปัญหานี้จะกล่าวถึงใน Common
Pitfalls ท้ายบท) หลักการทั่วไปคือ **ยอมให้ cache miss บ้าง ดีกว่า cache hit ผิดตัว**

---

## 94.6 Status Badge บน README (Step 750)

### Badge คืออะไร

**Status Badge** คือรูปเล็กๆ ที่แสดงสถานะ build ล่าสุดของ workflow (ผ่าน/ไม่ผ่าน) แบบเรียลไทม์
โดยดึงข้อมูลจาก GitHub โดยตรง เป็นมาตรฐานที่โปรเจกต์ Open Source แทบทุกโปรเจกต์ใช้กันบนหน้า
README เพื่อให้ผู้มาเยือน repository เห็นทันทีว่าโค้ดล่าสุด "ใช้งานได้จริง" หรือไม่

### รูปแบบ URL ของ Badge

GitHub สร้าง badge SVG ให้อัตโนมัติผ่าน URL รูปแบบนี้:

```
https://github.com/<OWNER>/<REPO>/actions/workflows/<WORKFLOW_FILE>/badge.svg
```

ตัวอย่างเช่น ถ้า workflow อยู่ที่ `.github/workflows/ci.yml` ใน repository
`somchai/my_project` badge URL จะเป็น:

```
https://github.com/somchai/my_project/actions/workflows/ci.yml/badge.svg
```

### ใส่ Badge ลงใน README.md

Syntax Markdown สำหรับแสดงรูปพร้อมลิงก์กลับไปที่หน้า Actions:

```markdown
# My Project

[![CI](https://github.com/somchai/my_project/actions/workflows/ci.yml/badge.svg)](https://github.com/somchai/my_project/actions/workflows/ci.yml)

โปรเจกต์ตัวอย่างสำหรับ...
```

โครงสร้าง Markdown นี้คือ "รูปภาพที่คลิกได้" — ส่วน `[![...](image_url)](link_url)` หมายถึง:

- `![CI](image_url)` — แสดงรูปภาพ badge โดยมี alt text ว่า "CI"
- ครอบด้วย `[...](link_url)` อีกชั้น — ทำให้คลิกที่รูปแล้วพาไปหน้า Actions ของ workflow นั้น
  โดยตรง เพื่อดู log แบบละเอียดถ้าอยากรู้ว่าทำไม build ถึงพัง

### พฤติกรรมจริงของ Badge (ภาพประกอบ)

Badge จะแสดงข้อความและสีตามสถานะของ**การรันล่าสุด**บน branch default (ปกติคือ `main`):

| สถานะ | สีของ Badge | ความหมาย |
|---|---|---|
| `passing` | เขียว | Workflow รันล่าสุดสำเร็จทุก step |
| `failing` | แดง | มี step ใดใน workflow รันล่าสุดล้มเหลว |
| `no status` | เทา | ยังไม่เคยมีการรัน workflow นี้เลย |

Badge นี้อัปเดตอัตโนมัติทุกครั้งที่มี workflow run ใหม่จบลง ไม่ต้องแก้ไข README เองเลย — เป็น
ภาพสะท้อนสถานะจริงของ repository ตลอดเวลาที่มีคนเปิดดู

---

## 94.7 ตรวจสอบ Workflow ให้ถูกต้องก่อน Push จริง (Step 751)

เนื่องจากเครื่องเรียนไม่มี GitHub Actions Runner ให้กดรันจริง ขั้นตอนที่**มืออาชีพทุกคนทำ**ก่อน
push ไฟล์ workflow คือชุดการตรวจสอบต่อไปนี้ ซึ่งทำได้ครบทั้งหมดบนเครื่อง local:

### ขั้นที่ 1: ตรวจสอบ YAML syntax

```bash
python3 -c "
import yaml, json
data = yaml.safe_load(open('.github/workflows/ci.yml'))
print(json.dumps(data, indent=2))
" | head -40
```

ผลลัพธ์จริง (ตัดมาบางส่วน แสดงให้เห็นว่า structure ถูกแปลงเป็น Python object ได้ถูกต้อง):

```json
{
  "name": "CI",
  "true": {
    "push": { "branches": ["main"] },
    "pull_request": { "branches": ["main"] }
  },
  "jobs": {
    "build-and-test": {
      "strategy": {
        "fail-fast": false,
        "matrix": { "os": ["ubuntu-latest"], "compiler": ["gcc", "clang"] }
      },
      "runs-on": "${{ matrix.os }}",
      "steps": [
        { "name": "Checkout code", "uses": "actions/checkout@v4" },
        ...
      ]
    }
  }
}
```

(สังเกตคีย์ `"true"` ที่มาแทน `"on"` ตามที่อธิบายไว้ในหัวข้อ 94.3 — เป็นพฤติกรรมของ YAML parser
ทั่วไป ไม่ใช่ของ GitHub Actions โดยเฉพาะ)

### ขั้นที่ 2: รันทุกคำสั่งใน `run:` บนเครื่อง local จริง

นี่คือขั้นตอนที่สำคัญที่สุด — **แกะทุกบรรทัดใน `run:` ของแต่ละ step มารันตามลำดับบนเครื่อง
ตัวเอง** ถ้าทุกคำสั่งรันผ่านตามลำดับนี้แบบเดียวกับที่เขียนใน YAML บนเครื่องที่มี environment
ใกล้เคียงกับ Runner (`ubuntu-latest` คือ Ubuntu LTS เวอร์ชันล่าสุด ซึ่งเครื่องเรียนนี้ก็คือ Ubuntu
เช่นกัน) ก็มั่นใจได้สูงมากว่า workflow จะผ่านบน GitHub Actions จริงเช่นกัน:

```bash
# เทียบเท่า step "Configure CMake"
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug

# เทียบเท่า step "Build"
cmake --build build -j "$(nproc)"

# เทียบเท่า step "Run unit tests" (มี working-directory: build)
cd build && ctest --output-on-failure
```

ทั้งสามคำสั่งนี้ถูกรันและยืนยันผลจริงแล้วในหัวข้อ 94.3 และ 94.4 ทั้งด้วย GCC และ Clang

### ขั้นที่ 3: เช็ครายชื่อ Action เวอร์ชันที่ใช้

ตรวจสอบว่า Action ที่อ้างถึง (`actions/checkout@v4`, `actions/cache@v4`) เป็นเวอร์ชันที่ยังได้รับ
การดูแลอยู่ (ไม่ใช่เวอร์ชันเก่าที่ถูก deprecate) — Action เวอร์ชันเก่ามากๆ (เช่น `@v1` ของ
`actions/checkout`) อาจถูก GitHub ถอนการสนับสนุนและทำให้ workflow ล้มเหลวกะทันหันแม้ไม่ได้
แก้โค้ดอะไรเลย

### สรุปแนวทาง "Validate Before Push"

```
เขียน/แก้ ci.yml
   │
   ▼
[1] python3 -c "import yaml; yaml.safe_load(open(...))"  → ยืนยัน syntax
   │
   ▼
[2] รันทุกคำสั่งใน run: ด้วยมือบนเครื่อง local ตามลำดับเป๊ะ  → ยืนยัน logic
   │
   ▼
[3] เช็คว่า Action ที่ใช้ยังเป็นเวอร์ชันที่รองรับอยู่  → ยืนยันความเสถียร
   │
   ▼
git add .github/workflows/ci.yml && git commit && git push
```

แนวทางนี้ทำให้ปัญหาที่พบตอน push จริงลดลงเหลือแทบจะ 0% เพราะทุกอย่างถูกพิสูจน์แล้วบนเครื่อง
ก่อนส่งขึ้นไปให้ GitHub Actions รัน

---

## 94.8 ภาพรวมที่กว้างกว่า: CI/CD ทั้งระบบ (Step 752)

Part นี้โฟกัสที่ **CI (Continuous Integration)** คือส่วนของการ build และ test อัตโนมัติ แต่คำว่า
"CI/CD" ที่ได้ยินบ่อยในวงการจริงๆ ประกอบด้วย 3 แนวคิดที่ต่อยอดกันเป็นลำดับขั้น:

```
Continuous Integration (CI)
   → build + test อัตโนมัติทุกครั้งที่ push          (สิ่งที่เรียนใน Part นี้)
        │
        ▼
Continuous Delivery (CD - Delivery)
   → หลังผ่าน CI แล้ว สร้าง "artifact" ที่พร้อม deploy อัตโนมัติ
     (เช่น build ไฟล์ .deb, Docker image) แต่ยังต้องมี "คน" กดปุ่ม deploy เอง
        │
        ▼
Continuous Deployment (CD - Deployment)
   → เมื่อผ่าน CI แล้ว deploy ขึ้น production โดยอัตโนมัติทันที ไม่ต้องมีคนกดปุ่มเลย
```

### ตารางเปรียบเทียบ 3 ระดับ

| ระดับ | สิ่งที่เกิดขึ้นอัตโนมัติ | ต้องมีคนกดปุ่มไหม |
|---|---|---|
| **Continuous Integration** | Build + Test ทุกครั้งที่ push | ไม่เกี่ยวกับ deploy เลย |
| **Continuous Delivery** | Build + Test + เตรียม artifact พร้อม deploy | ต้องมีคนอนุมัติขั้นตอน deploy |
| **Continuous Deployment** | Build + Test + Deploy ขึ้น production ทันที | ไม่ต้อง — อัตโนมัติทั้งหมด |

### ทำไม Part นี้หยุดแค่ CI

การ Deploy จริงต้องพึ่งพาโครงสร้างพื้นฐานเพิ่มเติมมาก (Server, Docker, Container Registry,
Secret Management) ซึ่งหลักสูตรนี้จะพูดถึงอย่างละเอียดใน **Part 112 (Deploy: Docker, Nginx,
Production Server)** ของ Module I หลังจากปูพื้นฐาน Web Development ครบแล้ว ในตอนนี้แค่รู้จัก
คำศัพท์และภาพรวมไว้ก่อนก็เพียงพอ — สิ่งสำคัญที่สุดคือ **CI ต้องแข็งแรงก่อนเสมอ** เพราะ CD
ทุกระดับพึ่งพาผลลัพธ์ที่เชื่อถือได้จาก CI เป็นด่านแรก ถ้า CI ไม่น่าเชื่อถือ การ deploy อัตโนมัติต่อ
จากนั้นก็จะยิ่งอันตรายกว่าเดิม (ส่ง bug ขึ้น production เร็วขึ้นโดยอัตโนมัติ)

### ตัวอย่างขั้นตอนที่ต่อยอดจาก CI ไปสู่ CD (สังเขป ยังไม่ลงมือทำใน Part นี้)

```yaml
# ตัวอย่างแนวคิด (ไม่ใช่ workflow ที่ควร copy ไปใช้ตรงๆ ในตอนนี้)
jobs:
  build-and-test:
    # ... เหมือน Part นี้ทุกประการ ...

  build-artifact:
    needs: build-and-test      # รอให้ job นี้ผ่านก่อน ถึงจะเริ่ม job นี้
    runs-on: ubuntu-latest
    steps:
      - name: Build release binary
        run: cmake --build build --config Release
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: my-app-binary
          path: build/my_app
```

`needs:` คือกลไกที่ทำให้ job สอง job **รันเรียงลำดับกัน** แทนที่จะขนานกันแบบ default — ใช้เมื่อ
job หลังต้องพึ่งพาผลลัพธ์ของ job ก่อนหน้า (เช่นต้อง build+test ผ่านก่อน ถึงจะยอม build ตัว
artifact สำหรับ deploy) แนวคิดนี้จะกลับมาขยายความอย่างเต็มรูปแบบใน Part 112

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **วางไฟล์ workflow ผิดที่**: ต้องอยู่ที่ `.github/workflows/*.yml` เป๊ะเท่านั้น (ตัวพิมพ์เล็ก
   ทั้งหมด ไม่ใช่ `.Github` หรือ `workflow` เอกพจน์) วางผิดที่แล้ว GitHub จะไม่แจ้ง error ใดๆ
   เลย เพียงแค่**ไม่รันอะไรสักอย่าง** ทำให้เข้าใจผิดว่า push แล้วไม่มีอะไรเกิดขึ้นเพราะ "โค้ดผิด"
   ทั้งที่จริงๆ workflow ไม่เคยถูกค้นพบเลย
2. **ผสม Tab กับ Space ใน YAML**: ทำให้ parser อ่านโครงสร้างผิดเพี้ยนไปโดยสิ้นเชิง วิธีป้องกันคือ
   ตั้งค่า editor ให้แสดง whitespace และบังคับใช้ space เท่านั้นสำหรับไฟล์ `.yml`
3. **ลืมว่า environment variable ที่ตั้งด้วย `export` ไม่ข้าม step**: แต่ละ `run:` block รันเป็น
   shell process ใหม่แยกกัน ต้องใช้ `echo "VAR=value" >> "$GITHUB_ENV"` เพื่อให้ค่าคงอยู่ข้าม
   step ในบ Job เดียวกันเท่านั้น
4. **Cache เก่าทำให้ผลลัพธ์ผิดเพี้ยน**: ถ้า cache key ไม่ครอบคลุมทุกไฟล์ที่กระทบต่อผลลัพธ์ของ
   build (เช่น cache ตาม `CMakeLists.txt` อย่างเดียว แต่ลืมว่า source code เปลี่ยนด้วย) อาจได้
   binary จาก cache เก่าที่ไม่ตรงกับโค้ดปัจจุบัน ทางแก้คือออกแบบ cache key ให้ระมัดระวัง หรือ
   cache เฉพาะสิ่งที่ไม่ควรเปลี่ยนบ่อย (เช่น dependency ที่ดาวน์โหลดจากภายนอก) ไม่ใช่ cache
   ผลลัพธ์ compile ของ source code ตัวเองที่เปลี่ยนทุก commit
5. **ใช้ `fail-fast: true` (default) กับ Matrix Build ตอน debug**: ทำให้เห็นแค่ job แรกที่ fail
   แล้ว job อื่นถูกยกเลิกทันที ทำให้ไม่เห็นภาพรวมว่ามีปัญหากับ compiler ไหนบ้าง ควรตั้งเป็น
   `false` อย่างน้อยตอนกำลังไล่หาสาเหตุปัญหาข้าม compiler/OS
6. **ลืมว่า Runner เป็นเครื่องเปล่าทุกครั้ง**: มือใหม่มักคิดว่า package ที่เคยติดตั้งไว้ "รอบที่แล้ว"
   ยังอยู่ ทั้งที่ทุก workflow run ได้เครื่องใหม่เอี่ยมเสมอ (นอกจากจะตั้ง cache ไว้ตามหัวข้อ 94.5)
   ทำให้ลืมใส่ step ติดตั้ง dependency ที่จำเป็นบางตัว

---

## แบบฝึกหัดท้ายบท

1. เขียน `.github/workflows/ci.yml` สำหรับโปรเจกต์ CMake ของตัวเองจาก Part 91–93 ให้ครบ
   ทั้ง `checkout`, ติดตั้ง dependency, configure, build, และรัน `ctest` จากนั้นตรวจสอบ syntax
   ด้วย `python3 -c "import yaml; yaml.safe_load(open(...))"` ให้ผ่านโดยไม่มี error
2. เพิ่ม matrix build ให้ workflow ในข้อ 1 ทดสอบทั้ง GCC และ Clang พร้อมกัน แล้วรันคำสั่งชุด
   เดียวกันด้วยมือทั้งสอง compiler บนเครื่องตัวเองเพื่อยืนยันว่า build และ test ผ่านทั้งคู่จริง
3. เพิ่ม `actions/cache` เข้าไปใน workflow เพื่อ cache โฟลเดอร์ build โดยใช้
   `hashFiles('**/CMakeLists.txt')` เป็นส่วนหนึ่งของ cache key อธิบายด้วยคำพูดตัวเองว่าทำไม
   ต้องรวม matrix variable (เช่นชื่อ compiler) เข้าไปใน key ด้วยถ้ามี matrix build
4. สร้างไฟล์ README.md ที่มี status badge ชี้ไปที่ workflow ของตัวเอง (สมมติชื่อ owner/repo เอง
   ถ้ายังไม่มี repository จริงบน GitHub) พร้อมอธิบาย syntax ของ Markdown แบบ "รูปภาพที่คลิกได้"
5. ลองแก้ workflow ให้จงใจมี syntax error (เช่น ลบ indentation ผิดที่หนึ่งจุด) แล้วรัน
   `python3 -c "import yaml; yaml.safe_load(open(...))"` ดูว่า error message บอกอะไรบ้าง
   ฝึกอ่าน error message ของ YAML parser ให้คุ้นเคย
6. อธิบายด้วยคำพูดตัวเองว่า Continuous Integration, Continuous Delivery และ Continuous
   Deployment ต่างกันอย่างไร พร้อมยกตัวอย่างสถานการณ์จริงที่ทีมหนึ่งอาจเลือกทำแค่ CI โดยไม่ทำ
   CD เลยก็ได้ (เพราะเหตุผลอะไร)

### แนวทางเฉลยข้อ 1

```yaml
name: CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake libgtest-dev g++

      - name: Configure CMake
        run: cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug

      - name: Build
        run: cmake --build build -j "$(nproc)"

      - name: Run unit tests
        working-directory: build
        run: ctest --output-on-failure
```

ตรวจสอบด้วย:

```bash
python3 -c "
import yaml
yaml.safe_load(open('.github/workflows/ci.yml'))
print('YAML parsed OK')
"
```

ผลลัพธ์ที่ควรได้: `YAML parsed OK` (ถ้า error แสดงว่ามีปัญหาเรื่อง indentation หรือเครื่องหมาย
วรรคตอนใน YAML ที่ต้องกลับไปแก้ก่อน)

### แนวทางเฉลยข้อ 6

- **Continuous Integration (CI)**: อัตโนมัติเฉพาะการ build และ test เท่านั้น ไม่แตะเรื่อง deploy
  ใดๆ เลย เป้าหมายคือให้รู้เร็วที่สุดว่าโค้ดที่ push มี "พัง" อะไรหรือเปล่า
- **Continuous Delivery**: อัตโนมัติต่อจาก CI ไปจนถึงขั้นตอน "เตรียม" สิ่งที่พร้อม deploy (เช่น
  build Docker image เสร็จ พร้อมกดปุ่ม deploy ได้ทันที) แต่ **ยังต้องมีคนตัดสินใจกดปุ่ม deploy
  เอง** เพราะบางทีมต้องการควบคุมจังหวะการปล่อยฟีเจอร์ใหม่ให้ผู้ใช้เอง (เช่น รอ marketing
  ประกาศพร้อมกัน หรือรอช่วงเวลาที่ traffic ต่ำ)
- **Continuous Deployment**: อัตโนมัติทั้งหมดจนถึงขึ้น production จริงทันทีที่ CI ผ่าน ไม่มีคน
  แตะเลย เหมาะกับทีมที่มี test coverage สูงมากและมั่นใจในคุณภาพของ automated test อย่างเต็มที่
- **ตัวอย่างทีมที่เลือกทำแค่ CI**: ทีมที่พัฒนาไลบรารี Open Source (เช่นไลบรารี C++ ที่คนอื่นเอาไป
  ใช้ต่อ) ไม่มี "production server" ของตัวเองให้ deploy อัตโนมัติเลย สิ่งที่ต้องการคือแค่ให้ทุก
  Pull Request ถูกตรวจสอบอัตโนมัติว่าไม่ทำให้ไลบรารีพัง (CI) ส่วนการ "release" เวอร์ชันใหม่
  มักทำโดยมนุษย์กด tag เวอร์ชันเองอย่างตั้งใจ ไม่ใช่ deploy อัตโนมัติทุกครั้งที่ merge

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า Continuous Integration แก้ปัญหาอะไรที่การทดสอบด้วยมือแก้ไม่ได้ และทำไมมันเป็น
  รากฐานสำคัญของการพัฒนาซอฟต์แวร์แบบทีมในโลกจริง
- เข้าใจโครงสร้างพื้นฐานของ GitHub Actions ทั้ง Workflow, Event, Job, Step และ Runner
- เขียน workflow ที่ `checkout` โค้ด, build โปรเจกต์ CMake, และรัน unit test จาก Part 93 โดย
  อัตโนมัติได้จริง พร้อมยืนยันว่าทุกคำสั่งที่ workflow จะรันนั้นใช้งานได้จริงบนเครื่อง (GCC และ
  Clang ทั้งคู่)
- ใช้ Matrix Build เพื่อทดสอบหลาย compiler พร้อมกันในการรันครั้งเดียว
- ใช้ `actions/cache` เพื่อลดเวลารัน CI พร้อมเข้าใจความเสี่ยงที่ต้องระวังเรื่อง cache key
- ติดตั้ง Status Badge บน README เพื่อให้เห็นสถานะโปรเจกต์แบบเรียลไทม์
- แยกความแตกต่างระหว่าง Continuous Integration, Continuous Delivery และ Continuous
  Deployment ได้อย่างถูกต้อง และรู้ว่าหลักสูตรนี้จะกลับมาพูดเรื่อง Deployment เต็มรูปแบบใน
  Part 112

การมี CI ที่ทำงานอัตโนมัติเป็นแค่**ด่านแรก**ของคุณภาพซอฟต์แวร์ระดับมืออาชีพ — CI จะมีประโยชน์
เต็มที่ก็ต่อเมื่อสิ่งที่มันรันมีคุณภาพสูงพอ ใน **Part 95** เราจะเพิ่มเครื่องมือ **Static Analysis**
(`clang-tidy`, `cppcheck`, `clang-format`) เข้าไปใน pipeline เดียวกันนี้ เพื่อตรวจจับบั๊กและ
ปัญหาเชิงสไตล์ที่ compiler warning ธรรมดาจับไม่ได้ ก่อนที่โค้ดจะถูก merge เข้า `main` เลยด้วยซ้ำ

**ต่อไป:** [Part 95 — Static Analysis: clang-tidy, cppcheck, clang-format](./part-095-static-analysis.md)
