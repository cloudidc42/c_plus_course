# Part 93: Unit Testing ด้วย Google Test และ Catch2 (Step 737–744)

> Module H — Build Systems, Testing และ Tooling | Part 93 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 737–744
> Part ก่อนหน้า: [Part 92 — Package Management: Conan และ vcpkg](./part-092-package-management.md) | Part ถัดไป: Part 94 — Continuous Integration สำหรับ C++ ด้วย GitHub Actions

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม Unit Test ถึงสำคัญมากในการพัฒนาซอฟต์แวร์ระดับมืออาชีพ และอธิบาย
   หลักการ **Arrange-Act-Assert** ที่ใช้เป็นโครงร่างของ Test Case ที่ดีทุกตัว
2. เขียน Test Case แรกด้วย Google Test ผ่านมาโคร `TEST()` และเข้าใจความแตกต่างระหว่าง
   Assertion ตระกูล `ASSERT_*` กับ `EXPECT_*` อย่างถ่องแท้ผ่านการทดลองจริง
3. เขียน Test Fixture ด้วย `TEST_F` สำหรับกรณีที่หลาย Test Case ต้องใช้ข้อมูลตั้งต้นร่วมกัน
4. Build และ Link โปรเจกต์ที่ใช้ Google Test เข้ากับ CMake ได้ทั้งผ่าน `find_package(GTest)`
   และเข้าใจข้อจำกัดของ `FetchContent` เมื่อไม่มี network เชื่อมต่อ
5. เขียน Test Case ด้วย Catch2 ผ่าน `TEST_CASE`, `REQUIRE`, และแบ่ง sub-test ด้วย `SECTION`
6. เปรียบเทียบ Google Test กับ Catch2 ได้อย่างชัดเจน และตัดสินใจเลือกใช้เครื่องมือที่เหมาะสม
   กับสถานการณ์ของโปรเจกต์จริงได้
7. เขียนชุด Unit Test ที่สมบูรณ์ให้กับฟังก์ชัน Sorting Algorithm จาก **Part 23** (พอร์ตมาเป็น
   C++ บน `std::vector`) ครอบคลุมทั้งกรณีปกติและ Edge Case

---

## 93.1 ทำไม Unit Test ถึงสำคัญ (Step 737)

ตลอด 92 Part ที่ผ่านมา เราตรวจสอบว่าโค้ดทำงานถูกต้องด้วยวิธีเดียวกันมาตลอด: เขียน `main()`
ที่เรียกฟังก์ชัน แล้ว `printf` ผลลัพธ์ออกมาดูด้วยตา (Manual Testing) วิธีนี้ใช้ได้ดีตอนเรียนรู้
concept ใหม่ๆ แต่มีปัญหาใหญ่เมื่อนำไปใช้กับโปรเจกต์ขนาดจริงที่มีฟังก์ชันหลายร้อยตัวและถูกแก้ไข
อยู่ตลอดเวลา:

1. **ตรวจสอบซ้ำด้วยมือไม่ scale**: ถ้าโปรเจกต์มี 200 ฟังก์ชัน การไล่ตรวจผลลัพธ์ด้วยตาทุกครั้ง
   ที่แก้โค้ดเป็นไปไม่ได้ในทางปฏิบัติ ยิ่งโปรเจกต์โตขึ้น ภาระนี้ยิ่งหนักขึ้นแบบทวีคูณ
2. **Regression**: เมื่อแก้บั๊กหรือเพิ่มฟีเจอร์ใหม่ในฟังก์ชัน A บางครั้งกลับไปทำให้ฟังก์ชัน B ที่
   เคยทำงานถูกต้องพังโดยไม่ตั้งใจ (เพราะ B เรียกใช้ A อยู่ภายใน) ถ้าไม่มีระบบตรวจสอบอัตโนมัติ
   ครอบคลุมทั้งโปรเจกต์ บั๊กแบบนี้มักไปโผล่ที่ผู้ใช้จริงแทนที่จะถูกจับตั้งแต่ตอนพัฒนา
3. **ไม่กล้า Refactor**: Refactoring (ปรับปรุงโครงสร้างโค้ดโดยไม่เปลี่ยนพฤติกรรมภายนอก) เป็น
   สิ่งจำเป็นสำหรับโค้ดที่มีอายุยืน แต่ถ้าไม่มี Test ที่เชื่อถือได้ นักพัฒนาจะกลัวการ Refactor
   เพราะไม่มีทางรู้แน่ชัดว่าการเปลี่ยนแปลงนั้นทำให้อะไรพังไปหรือไม่ ผลคือโค้ดสะสมความสกปรก
   (Technical Debt) ไปเรื่อยๆ โดยไม่มีใครกล้าแตะ

**Unit Test** คือการเขียนโค้ดขนาดเล็กที่ตรวจสอบว่า "หน่วยที่เล็กที่สุดของโปรแกรม" (มักหมายถึง
ฟังก์ชันเดียวหรือ class เดียว) ทำงานถูกต้องตามที่คาดหวัง **โดยอัตโนมัติ** และสามารถรันซ้ำได้
ทุกครั้งที่ต้องการในเวลาไม่กี่วินาที ต่างจาก Manual Testing ที่ต้องใช้แรงคนทุกครั้ง

### หลักการ Arrange-Act-Assert (AAA)

Test Case ที่ดีเกือบทั้งหมดมีโครงสร้าง 3 ส่วนเสมอ:

```
Arrange  →  เตรียมข้อมูล/สภาวะแวดล้อมที่จำเป็นก่อนทดสอบ
Act      →  เรียกใช้ฟังก์ชัน/โค้ดที่ต้องการทดสอบ
Assert   →  ตรวจสอบว่าผลลัพธ์ที่ได้ตรงกับที่คาดหวังหรือไม่
```

ตัวอย่างแนวคิด (ยังไม่ใช่ syntax ของ framework ใดๆ เพื่อให้เห็นภาพหลักการล้วนๆ ก่อน):

```
Arrange: สร้าง vector ที่มีข้อมูลไม่เรียงลำดับ {8, 3, 5, 4, 9, 1}
Act:     เรียกฟังก์ชัน sort(vector นั้น)
Assert:  ตรวจสอบว่า vector กลายเป็น {1, 3, 4, 5, 8, 9} จริง
```

โครงสร้างนี้ทำให้ Test Case อ่านง่าย คาดเดาได้ และไม่สับสนว่าส่วนไหนเป็นการเตรียมข้อมูล
ส่วนไหนเป็นการตรวจสอบผลลัพธ์จริง เราจะเห็นโครงสร้างนี้ปรากฏชัดเจนในทุกตัวอย่างโค้ดของ
Part นี้ (มักเขียนกำกับด้วย comment `// Arrange`, `// Act`, `// Assert` ตรงๆ)

### เครื่องมือ 2 ตัวที่จะเรียนใน Part นี้

C++ มี Unit Testing Framework ให้เลือกหลายตัว แต่สองตัวที่ได้รับความนิยมสูงสุดในอุตสาหกรรม
ปัจจุบัน (2026) คือ:

- **Google Test (GTest)**: พัฒนาโดย Google ใช้ในโปรเจกต์ขนาดใหญ่ระดับโลกมากมาย (Chromium,
  TensorFlow, LLVM บางส่วน) มี feature ครบครันมากรวมถึง Mocking (Google Mock) และ Death
  Test
- **Catch2**: Header-only Framework (ในรุ่นเก่า) ที่เน้นความเรียบง่ายและ syntax ที่อ่านง่ายเป็น
  ธรรมชาติ ได้รับความนิยมสูงมากในโปรเจกต์ Open Source ขนาดกลาง-เล็ก

ทั้งสองตัวติดตั้งพร้อมใช้งานบนเครื่องที่ใช้เขียนบทเรียนนี้แล้ว (`libgtest-dev` เวอร์ชัน 1.14.0
และ `catch2` เวอร์ชัน 3.4.0) เราจะเขียนและ**รันจริง**ทั้งสองเครื่องมือในบทนี้

---

## 93.2 Google Test: Test Case แรกด้วย TEST() (Step 738)

### ติดตั้งและตรวจสอบ Google Test

```bash
sudo apt install libgtest-dev -y
```

> **หมายเหตุสำคัญเกี่ยวกับ Ubuntu กับ libgtest-dev**: ในอดีต (Ubuntu หลายเวอร์ชันก่อนหน้า)
> package `libgtest-dev` แจกมาแค่ **source code** (ไว้ที่ `/usr/src/googletest/`) โดยไม่มี
> ไฟล์ `.a`/`.so` ที่ build เสร็จแล้วให้เลย ผู้ใช้ต้อง `cmake`/`make` เองจาก
> `/usr/src/googletest` แล้ว copy ไฟล์ `.a` ไปไว้ที่ `/usr/lib` ด้วยมือ — เป็นขั้นตอนที่โด่งดัง
> ว่าสร้างความสับสนให้ผู้เริ่มต้นมานาน อย่างไรก็ตาม บนเครื่องที่ใช้เขียนบทเรียนนี้ (Ubuntu 24.04,
> `libgtest-dev` 1.14.0) เราตรวจสอบพบว่า package **ได้ build และติดตั้ง static library
> สำเร็จรูปไว้ให้แล้ว** พร้อมทั้งไฟล์ CMake Config File (`GTestConfig.cmake`) ครบถ้วน:
> ```
> $ find / -iname "libgtest*.a" 2>/dev/null
> /usr/lib/x86_64-linux-gnu/libgtest_main.a
> /usr/lib/x86_64-linux-gnu/libgtest.a
> $ find / -iname "GTestConfig.cmake" 2>/dev/null
> /usr/lib/x86_64-linux-gnu/cmake/GTest/GTestConfig.cmake
> ```
> ดังนั้นในเครื่องนี้ `find_package(GTest)` จะทำงานได้ทันทีโดยไม่ต้อง build เองเพิ่ม (จะพิสูจน์
> ในหัวข้อ 93.6) แต่ถ้าใช้เครื่อง Ubuntu เวอร์ชันเก่ากว่าที่ยังไม่มี CMake Config File สำเร็จรูป
> วิธีแก้มาตรฐานคือใช้ CMake `FetchContent` ดาวน์โหลดและ build Google Test เองจาก GitHub
> (จะสาธิตและอธิบายข้อจำกัดในหัวข้อ 93.6 เช่นกัน)

### Test Case แรก

```cpp
#include <gtest/gtest.h>

TEST(BasicMath, AdditionWorks) {
    // Arrange
    int a = 2;
    int b = 3;

    // Act
    int result = a + b;

    // Assert
    EXPECT_EQ(result, 5);
}
```

โครงสร้างของมาโคร `TEST(กลุ่ม, ชื่อเทส)`:

- **กลุ่ม (Test Suite Name)**: `BasicMath` — ใช้จัดกลุ่ม Test Case ที่เกี่ยวข้องกันไว้ด้วยกัน
  (มักตั้งชื่อตาม class/module ที่กำลังทดสอบ)
- **ชื่อเทส (Test Name)**: `AdditionWorks` — อธิบายสั้นๆ ว่า Test Case นี้ตรวจสอบอะไร ควรตั้ง
  ให้สื่อความหมายชัดเจน (ไม่ใช่ `Test1`, `Test2` ที่ไม่บอกอะไรเลย)

Compile และ Link (ต้อง `-lgtest -lgtest_main -lpthread` เพราะ Google Test ใช้ threading
ภายในเช่นเดียวกับ Google Benchmark ที่เรียนใน Part 90):

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 test_basic.cpp -o test_basic -lgtest -lgtest_main -lpthread
./test_basic
```

ผลลัพธ์จริง:

```
Running main() from ./googletest/src/gtest_main.cc
[==========] Running 1 test from 1 test suite.
[----------] Global test environment set-up.
[----------] 1 test from BasicMath
[ RUN      ] BasicMath.AdditionWorks
[       OK ] BasicMath.AdditionWorks (0 ms)
[----------] 1 test from BasicMath (0 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test suite ran. (0 ms total)
[  PASSED  ] 1 test.
```

สังเกตว่าเราไม่ได้เขียนฟังก์ชัน `main()` เองเลย — ไฟล์ `libgtest_main.a` ที่ link เข้าไปมีฟังก์ชัน
`main()` สำเร็จรูปให้แล้ว (ทำหน้าที่ค้นหาและรัน `TEST()` ทุกตัวที่ลงทะเบียนไว้ในโปรแกรม พิมพ์
รายงานผล และคืนค่า exit code `0` ถ้าผ่านหมด หรือค่าอื่นถ้ามี test ที่ล้มเหลว — เชื่อมโยงกับความรู้
เรื่อง Exit Code จาก Part 1 โดยตรง)

---

## 93.3 ASSERT กับ EXPECT ต่างกันอย่างไร (Step 739)

Google Test มี Assertion 2 ตระกูลหลักที่ทำหน้าที่คล้ายกันแต่พฤติกรรมต่างกันอย่างสำคัญ:

| ตระกูล | พฤติกรรมเมื่อ assertion ล้มเหลว |
|---|---|
| `EXPECT_*` | บันทึกว่าล้มเหลว แต่**ทำงานต่อ**ในบรรทัดถัดไปของ Test Case นั้น |
| `ASSERT_*` | บันทึกว่าล้มเหลว และ**หยุด Test Case นั้นทันที** (คล้าย `return` ออกจากฟังก์ชัน) |

### พิสูจน์ความต่างด้วยการรันจริง

```cpp
#include <gtest/gtest.h>
#include <cstdio>

TEST(AssertVsExpect, UsingExpectContinuesAfterFailure) {
    EXPECT_EQ(1 + 1, 3);          // ผิด แต่ EXPECT ไม่หยุด -- โค้ดด้านล่างยังรันต่อ
    printf("บรรทัดนี้ยังถูกพิมพ์ เพราะ EXPECT ไม่ทำให้ test หยุดทำงาน\n");
    EXPECT_EQ(2 + 2, 4);          // ถูก
}

TEST(AssertVsExpect, UsingAssertStopsImmediately) {
    ASSERT_EQ(1 + 1, 3);          // ผิด และ ASSERT หยุดทันที
    printf("บรรทัดนี้จะไม่ถูกพิมพ์เลย เพราะ ASSERT หยุด test ไปแล้ว\n");
    ASSERT_EQ(2 + 2, 4);
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 test_assert_vs_expect.cpp -o test_assert_vs_expect \
    -lgtest -lgtest_main -lpthread
./test_assert_vs_expect
```

ผลลัพธ์จริง:

```
Running main() from ./googletest/src/gtest_main.cc
[==========] Running 2 tests from 1 test suite.
[----------] Global test environment set-up.
[----------] 2 tests from AssertVsExpect
[ RUN      ] AssertVsExpect.UsingExpectContinuesAfterFailure
test_assert_vs_expect.cpp:4: Failure
Expected equality of these values:
  1 + 1
    Which is: 2
  3

บรรทัดนี้ยังถูกพิมพ์ เพราะ EXPECT ไม่ทำให้ test หยุดทำงาน
[  FAILED  ] AssertVsExpect.UsingExpectContinuesAfterFailure (0 ms)
[ RUN      ] AssertVsExpect.UsingAssertStopsImmediately
test_assert_vs_expect.cpp:10: Failure
Expected equality of these values:
  1 + 1
    Which is: 2
  3

[  FAILED  ] AssertVsExpect.UsingAssertStopsImmediately (0 ms)
[----------] 2 tests from AssertVsExpect (0 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 0 tests.
[  FAILED  ] 2 tests, listed below:
[  FAILED  ] AssertVsExpect.UsingExpectContinuesAfterFailure
[  FAILED  ] AssertVsExpect.UsingAssertStopsImmediately

 2 FAILED TESTS
```

หลักฐานชัดเจนจากผลลัพธ์จริง: ใน `UsingExpectContinuesAfterFailure` บรรทัด
`printf("บรรทัดนี้ยังถูกพิมพ์...")` **ถูกพิมพ์ออกมาจริง** แม้ `EXPECT_EQ` ก่อนหน้าจะล้มเหลว
ในขณะที่ `UsingAssertStopsImmediately` **ไม่มีข้อความ printf ปรากฏเลย** เพราะ `ASSERT_EQ`
หยุดการทำงานของ Test Case นั้นทันทีที่ล้มเหลว

### เมื่อไหร่ควรใช้ตัวไหน

> **กฎทองของ Part นี้**: ใช้ `ASSERT_*` เมื่อผลลัพธ์ของบรรทัดนั้นเป็น**เงื่อนไขที่ต้องผ่านก่อน
> ถึงจะทดสอบบรรทัดถัดไปได้อย่างปลอดภัย** (เช่น ตรวจสอบว่า pointer ไม่ใช่ `nullptr` ก่อนจะ
> dereference มันในบรรทัดถัดไป) ใช้ `EXPECT_*` เมื่อต้องการตรวจสอบหลายเงื่อนไขที่**เป็นอิสระ
> จากกัน**ในฟังก์ชันเดียว เพื่อให้เห็นภาพรวมว่ามีกี่จุดที่ผิดพลาดในการรันครั้งเดียว แทนที่จะเห็น
> แค่จุดแรกแล้วต้องแก้ทีละจุดวนไปเรื่อยๆ

ตัวอย่างการใช้ `ASSERT_*` อย่างถูกต้องเพื่อป้องกัน Crash:

```cpp
TEST(PointerSafety, CheckBeforeDereference) {
    int* ptr = create_some_pointer();  // อาจคืน nullptr ถ้าเกิดข้อผิดพลาด
    ASSERT_NE(ptr, nullptr);           // ถ้า ptr เป็น nullptr ให้หยุดทันที
    EXPECT_EQ(*ptr, 42);               // บรรทัดนี้ dereference ptr ได้อย่างปลอดภัย
                                        // เพราะผ่าน ASSERT_NE มาแล้วเท่านั้น
}
```

ถ้าใช้ `EXPECT_NE` แทน `ASSERT_NE` ในตัวอย่างนี้ และ `ptr` เป็น `nullptr` จริง โปรแกรมจะไป
dereference `nullptr` ที่บรรทัดถัดไปทันที ทำให้ทั้งโปรแกรม test **Crash (Segmentation Fault)**
ไปเลยทั้งกระบวนการ ไม่ใช่แค่ Test Case นั้นล้มเหลวแบบสวยงาม — นี่คือเหตุผลเชิงเทคนิคที่แท้จริง
ว่าทำไมต้องแยกแยะการใช้งานทั้งสองตระกูลให้ถูกต้อง

### Assertion Macro ที่ใช้บ่อยที่สุด

| Macro | ตรวจสอบว่า |
|---|---|
| `ASSERT_EQ(a, b)` / `EXPECT_EQ(a, b)` | `a == b` |
| `ASSERT_NE(a, b)` / `EXPECT_NE(a, b)` | `a != b` |
| `ASSERT_TRUE(cond)` / `EXPECT_TRUE(cond)` | `cond` เป็นจริง |
| `ASSERT_FALSE(cond)` / `EXPECT_FALSE(cond)` | `cond` เป็นเท็จ |
| `ASSERT_LT/LE/GT/GE(a, b)` | `a < b`, `a <= b`, `a > b`, `a >= b` ตามลำดับ |
| `ASSERT_STREQ(s1, s2)` | เปรียบเทียบ C-string (`char*`) ด้วย `strcmp` ภายใน แทนการเทียบ pointer ตรงๆ |
| `ASSERT_THROW(stmt, ExceptionType)` | `stmt` throw exception ชนิดที่ระบุจริง (เชื่อมโยงกับ Part 54) |
| `ASSERT_NO_THROW(stmt)` | `stmt` ไม่ throw exception ใดๆ เลย |

---

## 93.4 Test Fixture ด้วย TEST_F (Step 740)

เมื่อหลาย Test Case ต้องใช้ข้อมูลตั้งต้นชุดเดียวกัน (เช่น ต้องสร้าง object ที่ setup ซับซ้อนก่อน
ทุกครั้ง) การเขียนโค้ด setup ซ้ำในทุก `TEST()` จะทำให้โค้ดซ้ำซ้อนมาก **Test Fixture** คือ class
ที่สืบทอดจาก `::testing::Test` มีเมธอด `SetUp()`/`TearDown()` ที่ถูกเรียกก่อน/หลัง**ทุก** Test
Case ที่ใช้ Fixture นั้นโดยอัตโนมัติ (แนวคิดเดียวกับ `benchmark::Fixture` ที่เรียนใน Part 90
หัวข้อ 90.7 ทุกประการ):

```cpp
#include <gtest/gtest.h>
#include <vector>

class SortFixtureTest : public ::testing::Test {
protected:
    void SetUp() override {
        data = {40, 10, 30, 20, 50};
    }
    std::vector<int> data;
};

TEST_F(SortFixtureTest, BubbleSortProducesAscendingOrder) {
    // ในตัวอย่างนี้ data ถูกเตรียมให้แล้วโดย SetUp() อัตโนมัติ ก่อนเข้าสู่ Test Case นี้
    std::sort(data.begin(), data.end());
    EXPECT_EQ(data, (std::vector<int>{10, 20, 30, 40, 50}));
}

TEST_F(SortFixtureTest, DataStartsWithFiveElements) {
    // Fixture ถูกสร้างใหม่ทุก Test Case -- data กลับมาเป็น {40, 10, 30, 20, 50} เสมอ
    // ไม่ถูกกระทบจาก Test Case อื่นที่รันก่อนหน้า
    EXPECT_EQ(data.size(), 5u);
}
```

จุดสำคัญที่ต้องเข้าใจ: **Google Test สร้าง Fixture object ขึ้นใหม่ทุกครั้งสำหรับแต่ละ Test
Case** (ไม่ใช้ object เดียวกันซ้ำข้าม Test Case) ทำให้แต่ละ Test Case ทำงานอย่างเป็นอิสระจาก
กันอย่างสมบูรณ์ (Test Isolation) — `DataStartsWithFiveElements` จะเห็น `data` เป็นค่าตั้งต้น
เสมอ ไม่ว่า `BubbleSortProducesAscendingOrder` จะรันก่อนหรือหลังก็ตาม และไม่ว่าจะรันตาม
ลำดับไหนก็ได้ผลเหมือนเดิม

ความแตกต่างระหว่าง `TEST()` กับ `TEST_F()` มีแค่จุดเดียว: `TEST_F` ต้องระบุชื่อ Fixture Class
แทนชื่อกลุ่มแบบอิสระ (`TEST_F(SortFixtureTest, ...)` ผูกกับ `class SortFixtureTest` โดยตรง)
ส่วน syntax ของ Assertion ภายในเหมือนกันทุกประการกับ `TEST()` ปกติ

---

## 93.5 พอร์ต Sorting Algorithm จาก Part 23 มาเป็น C++ (Step 744 เตรียมข้อมูล)

ก่อนจะเขียน Test Suite ที่สมบูรณ์ในหัวข้อถัดไป เราต้องมีฟังก์ชันที่จะทดสอบก่อน — พอร์ต
`bubble_sort` และ `quick_sort` จาก **Part 23** (ซึ่งเดิมเขียนเป็น C บน `int arr[]`) มาเป็น C++
บน `std::vector<int>&` เพื่อให้เข้ากับสไตล์ Modern C++ ที่ใช้ตลอด Module H เป็นต้นไป:

`sort_utils.hpp`:

```cpp
#pragma once

#include <vector>

namespace sortutils {

// พอร์ตจาก Part 23 (bubble_sort บน int arr[] แบบ C) มาเป็น C++ บน std::vector<int>&
void bubble_sort(std::vector<int>& v);

// พอร์ตจาก Part 23 (quick_sort/partition บน int arr[] แบบ C) มาเป็น C++ บน std::vector<int>&
void quick_sort(std::vector<int>& v);

}  // namespace sortutils
```

`sort_utils.cpp`:

```cpp
#include "sort_utils.hpp"

#include <utility>

namespace sortutils {

void bubble_sort(std::vector<int>& v) {
    const std::size_t n = v.size();
    if (n < 2) {
        return;
    }
    for (std::size_t i = 0; i < n - 1; i++) {
        bool swapped = false;
        for (std::size_t j = 0; j < n - 1 - i; j++) {
            if (v[j] > v[j + 1]) {
                std::swap(v[j], v[j + 1]);
                swapped = true;
            }
        }
        if (!swapped) {
            break;
        }
    }
}

namespace {

// ใช้ index แบบ long (เหมือนต้นฉบับ Part 23 ทุกประการ) เพื่อให้ i = low - 1 ติดลบได้
// อย่างปลอดภัยโดยไม่เกิด unsigned integer underflow แบบที่จะเกิดถ้าใช้ std::size_t ตรงๆ
long partition_range(std::vector<int>& v, long low, long high) {
    int pivot = v[static_cast<std::size_t>(high)];
    long i = low - 1;

    for (long j = low; j < high; j++) {
        if (v[static_cast<std::size_t>(j)] < pivot) {
            i++;
            std::swap(v[static_cast<std::size_t>(i)], v[static_cast<std::size_t>(j)]);
        }
    }
    std::swap(v[static_cast<std::size_t>(i + 1)], v[static_cast<std::size_t>(high)]);
    return i + 1;
}

void quick_sort_range(std::vector<int>& v, long low, long high) {
    if (low < high) {
        long p = partition_range(v, low, high);
        quick_sort_range(v, low, p - 1);
        quick_sort_range(v, p + 1, high);
    }
}

}  // namespace

void quick_sort(std::vector<int>& v) {
    if (v.size() < 2) {
        return;
    }
    quick_sort_range(v, 0, static_cast<long>(v.size()) - 1);
}

}  // namespace sortutils
```

ก่อนเขียน Test Suite เต็มรูปแบบ ลองยืนยันด้วยโปรแกรมเล็กๆ ก่อนว่าฟังก์ชันทำงานถูกต้อง
(Manual Testing แบบเดิมที่ใช้มาตลอดหลักสูตร — ทำครั้งสุดท้ายก่อนเปลี่ยนไปใช้ Unit Test
อัตโนมัติแทนตั้งแต่หัวข้อถัดไป):

```cpp
#include <cstdio>
#include <vector>
#include "sort_utils.hpp"

int main() {
    std::vector<int> a = {8, 3, 5, 4, 9, 1};
    std::vector<int> b = a;
    sortutils::bubble_sort(a);
    sortutils::quick_sort(b);
    for (int x : a) printf("%d ", x);
    printf("\n");
    for (int x : b) printf("%d ", x);
    printf("\n");
    return 0;
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 smoke_main.cpp sort_utils.cpp -o smoke_main
./smoke_main
```

ผลลัพธ์จริง:

```
1 3 4 5 8 9
1 3 4 5 8 9
```

ทั้งสองฟังก์ชันเรียงข้อมูลถูกต้องตรงกัน ({1, 3, 4, 5, 8, 9} เหมือนกับผลลัพธ์ในต้นฉบับ Part 23)
พร้อมนำไปเขียน Test Suite อัตโนมัติในหัวข้อถัดไป

---

## 93.6 Build และ Link กับ CMake: find_package กับ FetchContent (Step 741)

### แนวทางที่ 1: find_package(GTest) — วิธีที่ใช้งานได้จริงบนเครื่องนี้

```cmake
cmake_minimum_required(VERSION 3.20)
project(SortUtilsTests CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

enable_testing()

add_library(sort_utils STATIC sort_utils.cpp)
target_include_directories(sort_utils PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

find_package(GTest REQUIRED)

add_executable(sort_utils_gtest test_sort_utils_gtest.cpp)
target_link_libraries(sort_utils_gtest PRIVATE sort_utils GTest::gtest GTest::gtest_main)

include(GoogleTest)
gtest_discover_tests(sort_utils_gtest)
```

Configure จริง:

```bash
mkdir build && cd build
cmake ..
```

```
-- The CXX compiler identification is GNU 13.3.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Found GTest: /usr/lib/x86_64-linux-gnu/cmake/GTest/GTestConfig.cmake (found version "1.14.0")
-- Configuring done (0.3s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/sortlib/build
```

`find_package(GTest REQUIRED)` หา CMake Config File ที่ `libgtest-dev` ติดตั้งไว้ให้เจอทันที
(บรรทัด `Found GTest: ... (found version "1.14.0")` ยืนยันชัดเจน) — เป็นวิธีเดียวกันเป๊ะกับที่
เรียนเรื่อง `find_package` ใน Part 92 หัวข้อ 92.5 ทุกประการ, `GTest::gtest` และ
`GTest::gtest_main` คือ Imported Target ที่ package นี้ประกาศไว้ให้

`include(GoogleTest)` + `gtest_discover_tests(...)` เป็นโมดูลเสริมที่มากับ CMake เอง ทำหน้าที่
สแกนหา `TEST()`/`TEST_F()` ทั้งหมดในไฟล์ executable แล้วลงทะเบียนแต่ละอันเป็น **CTest Test**
แยกกัน ทำให้สามารถรันทีละ Test Case ผ่าน `ctest` ได้ (จะเห็นในหัวข้อ 93.8)

### แนวทางที่ 2: FetchContent — วิธีมาตรฐานเมื่อไม่มี Config File สำเร็จรูป (แต่ใช้ไม่ได้ใน
sandbox นี้เพราะข้อจำกัด network)

ในกรณีที่เครื่องไม่มี `libgtest-dev` ติดตั้งไว้เลย (หรือ Ubuntu เวอร์ชันเก่าที่ไม่มี CMake Config
File สำเร็จรูปตามที่อธิบายในหัวข้อ 93.2) วิธีมาตรฐานที่นิยมที่สุดในปัจจุบันคือใช้
`FetchContent` ของ CMake ดาวน์โหลดและ build Google Test จาก GitHub โดยตรงตอน configure:

```cmake
cmake_minimum_required(VERSION 3.20)
project(SortUtilsTests CXX)

include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip
)
FetchContent_MakeAvailable(googletest)

add_executable(sort_utils_gtest test_sort_utils_gtest.cpp sort_utils.cpp)
target_link_libraries(sort_utils_gtest PRIVATE gtest_main)
```

เราทดลองรัน `cmake ..` กับไฟล์นี้จริงในสภาพแวดล้อมของบทเรียนนี้ ผลลัพธ์คือ **ล้มเหลว** ด้วย
เหตุผลเดียวกับที่เจอใน Part 92 (vcpkg/Conan ล้มเหลวเพราะ network ถูกจำกัด):

```
CMake Error at /usr/share/cmake-3.28/Modules/FetchContent.cmake:1679 (message):
  Build step for googletest failed: 2
Call Stack (most recent call first):
  /usr/share/cmake-3.28/Modules/FetchContent.cmake:1819:EVAL:2 (__FetchContent_directPopulate)
  /usr/share/cmake-3.28/Modules/FetchContent.cmake:1819 (cmake_language)
  /usr/share/cmake-3.28/Modules/FetchContent.cmake:2033 (FetchContent_Populate)
  CMakeLists.txt:8 (FetchContent_MakeAvailable)

-- Configuring incomplete, errors occurred!
```

การ debug เพิ่มเติมยืนยันว่าสาเหตุคือ CMake พยายาม `curl` ไฟล์ zip จาก `github.com` แล้วเจอ
ปัญหาการเชื่อมต่อ TLS ผ่าน proxy ของ sandbox เช่นเดียวกับที่ vcpkg/Conan เจอใน Part 92 —
**นี่คือเหตุผลตรงไปตรงมาที่บทเรียนนี้ใช้แนวทางที่ 1 (`find_package`) เป็นหลักในทุกตัวอย่างที่
ตามมา**: เพราะเป็นแนวทางเดียวที่รันได้จริงในสภาพแวดล้อมนี้ แต่ผู้เรียนควรรู้จัก syntax ของ
`FetchContent` ไว้ด้วย เพราะเป็นวิธีที่ใช้กันแพร่หลายมากในโปรเจกต์จริงที่มี network ปกติ
(โดยเฉพาะเมื่อต้องการ pin เวอร์ชันของ Google Test ให้ตรงกันทุกเครื่องแบบเดียวกับที่ vcpkg/
Conan ทำ โดยไม่ต้องพึ่งเวอร์ชันที่ OS แจกให้)

### รัน Test Suite เต็มรูปแบบ

กลับมาที่แนวทางที่ 1 ที่ใช้งานได้จริง — เขียน Test Suite เต็มรูปแบบครอบคลุมทั้ง `bubble_sort`
และ `quick_sort` รวมถึง Edge Case สำคัญ (ว่างเปล่า, สมาชิกเดียว, ข้อมูลซ้ำ, เรียงอยู่แล้ว):

```cpp
#include <gtest/gtest.h>
#include <vector>
#include "sort_utils.hpp"

TEST(BubbleSort, SortsUnorderedVector) {
    std::vector<int> v = {8, 3, 5, 4, 9, 1};
    sortutils::bubble_sort(v);
    ASSERT_EQ(v, (std::vector<int>{1, 3, 4, 5, 8, 9}));
}

TEST(BubbleSort, HandlesEmptyVector) {
    std::vector<int> v;
    sortutils::bubble_sort(v);
    EXPECT_TRUE(v.empty());
}

TEST(BubbleSort, HandlesSingleElement) {
    std::vector<int> v = {42};
    sortutils::bubble_sort(v);
    ASSERT_EQ(v.size(), 1u);
    EXPECT_EQ(v[0], 42);
}

TEST(BubbleSort, HandlesAlreadySorted) {
    std::vector<int> v = {1, 2, 3, 4, 5};
    sortutils::bubble_sort(v);
    EXPECT_EQ(v, (std::vector<int>{1, 2, 3, 4, 5}));
}

TEST(BubbleSort, HandlesDuplicates) {
    std::vector<int> v = {5, 3, 5, 1, 3};
    sortutils::bubble_sort(v);
    EXPECT_EQ(v, (std::vector<int>{1, 3, 3, 5, 5}));
}

TEST(QuickSort, SortsUnorderedVector) {
    std::vector<int> v = {8, 3, 5, 4, 9, 1};
    sortutils::quick_sort(v);
    ASSERT_EQ(v, (std::vector<int>{1, 3, 4, 5, 8, 9}));
}

TEST(QuickSort, HandlesEmptyVector) {
    std::vector<int> v;
    sortutils::quick_sort(v);
    EXPECT_TRUE(v.empty());
}

TEST(QuickSort, HandlesAlreadySorted) {
    std::vector<int> v = {1, 2, 3, 4, 5};
    sortutils::quick_sort(v);
    EXPECT_EQ(v, (std::vector<int>{1, 2, 3, 4, 5}));
}

TEST(QuickSort, HandlesReverseSorted) {
    std::vector<int> v = {5, 4, 3, 2, 1};
    sortutils::quick_sort(v);
    EXPECT_EQ(v, (std::vector<int>{1, 2, 3, 4, 5}));
}

class SortFixtureTest : public ::testing::Test {
protected:
    void SetUp() override {
        data = {40, 10, 30, 20, 50};
    }
    std::vector<int> data;
};

TEST_F(SortFixtureTest, BubbleSortProducesAscendingOrder) {
    sortutils::bubble_sort(data);
    EXPECT_EQ(data, (std::vector<int>{10, 20, 30, 40, 50}));
}

TEST_F(SortFixtureTest, QuickSortProducesAscendingOrder) {
    sortutils::quick_sort(data);
    EXPECT_EQ(data, (std::vector<int>{10, 20, 30, 40, 50}));
}
```

```bash
cmake --build .
./sort_utils_gtest
```

ผลลัพธ์จริง (build สำเร็จและ Test ทั้งหมดผ่าน):

```
[ 25%] Building CXX object CMakeFiles/sort_utils.dir/sort_utils.cpp.o
[ 50%] Linking CXX static library libsort_utils.a
[ 50%] Built target sort_utils
[ 75%] Building CXX object CMakeFiles/sort_utils_gtest.dir/test_sort_utils_gtest.cpp.o
[100%] Linking CXX executable sort_utils_gtest
[100%] Built target sort_utils_gtest
Running main() from ./googletest/src/gtest_main.cc
[==========] Running 11 tests from 3 test suites.
[----------] Global test environment set-up.
[----------] 5 tests from BubbleSort
[ RUN      ] BubbleSort.SortsUnorderedVector
[       OK ] BubbleSort.SortsUnorderedVector (0 ms)
[ RUN      ] BubbleSort.HandlesEmptyVector
[       OK ] BubbleSort.HandlesEmptyVector (0 ms)
[ RUN      ] BubbleSort.HandlesSingleElement
[       OK ] BubbleSort.HandlesSingleElement (0 ms)
[ RUN      ] BubbleSort.HandlesAlreadySorted
[       OK ] BubbleSort.HandlesAlreadySorted (0 ms)
[ RUN      ] BubbleSort.HandlesDuplicates
[       OK ] BubbleSort.HandlesDuplicates (0 ms)
[----------] 5 tests from BubbleSort (0 ms total)

[----------] 4 tests from QuickSort
[ RUN      ] QuickSort.SortsUnorderedVector
[       OK ] QuickSort.SortsUnorderedVector (0 ms)
[ RUN      ] QuickSort.HandlesEmptyVector
[       OK ] QuickSort.HandlesEmptyVector (0 ms)
[ RUN      ] QuickSort.HandlesAlreadySorted
[       OK ] QuickSort.HandlesAlreadySorted (0 ms)
[ RUN      ] QuickSort.HandlesReverseSorted
[       OK ] QuickSort.HandlesReverseSorted (0 ms)
[----------] 4 tests from QuickSort (0 ms total)

[----------] 2 tests from SortFixtureTest
[ RUN      ] SortFixtureTest.BubbleSortProducesAscendingOrder
[       OK ] SortFixtureTest.BubbleSortProducesAscendingOrder (0 ms)
[ RUN      ] SortFixtureTest.QuickSortProducesAscendingOrder
[       OK ] SortFixtureTest.QuickSortProducesAscendingOrder (0 ms)
[----------] 2 tests from SortFixtureTest (0 ms total)

[----------] Global test environment tear-down
[==========] 11 tests from 3 test suites ran. (0 ms total)
[  PASSED  ] 11 tests.
```

ทั้ง 11 Test Case ผ่านหมด — ยืนยันว่า `bubble_sort` และ `quick_sort` ที่พอร์ตมาจาก Part 23
ทำงานถูกต้องครอบคลุมทั้งกรณีปกติและ Edge Case สำคัญทั้งหมด

---

## 93.7 Catch2: TEST_CASE, REQUIRE, SECTION (Step 742)

**Catch2** ออกแบบมาให้ syntax กระชับและอ่านเป็นภาษาธรรมชาติมากกว่า Google Test อย่าง
ชัดเจน มาโครหลักที่ต้องรู้จัก:

| มาโคร | บทบาท |
|---|---|
| `TEST_CASE("คำอธิบาย", "[tag]")` | เทียบเท่ากับ `TEST()` ของ Google Test — นิยาม Test Case หนึ่งตัว |
| `REQUIRE(cond)` | เทียบเท่ากับ `ASSERT_TRUE`/`ASSERT_EQ` — ถ้าล้มเหลวหยุด Test Case ทันที |
| `CHECK(cond)` | เทียบเท่ากับ `EXPECT_TRUE`/`EXPECT_EQ` — ถ้าล้มเหลวยังทำงานต่อ |
| `SECTION("คำอธิบาย")` | แบ่ง Test Case เดียวออกเป็น sub-test หลายอันที่ใช้ Arrange ร่วมกัน |

### Test Case แรกด้วย Catch2

```cpp
#define CATCH_CONFIG_MAIN
#include <catch2/catch_all.hpp>

TEST_CASE("การบวกเลขพื้นฐานทำงานถูกต้อง", "[math]") {
    int a = 2;
    int b = 3;
    REQUIRE(a + b == 5);
}
```

Compile ผ่าน `pkg-config` (Catch2 เตรียม `.pc` file ไว้ให้ ทำให้ไม่ต้องจำ flag การ link เอง):

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 test_basic_catch2.cpp -o test_basic_catch2 \
    $(pkg-config --cflags --libs catch2-with-main)
./test_basic_catch2
```

ผลลัพธ์จริง:

```
Randomness seeded to: 642185387

===============================================================================
All tests passed (1 assertion in 1 test case)
```

### SECTION — จุดเด่นที่ทำให้ Catch2 กระชับกว่า Fixture ของ Google Test

`SECTION` ช่วยให้เขียนหลาย sub-test ที่ใช้ข้อมูลตั้งต้นร่วมกันได้โดยไม่ต้องประกาศ class Fixture
แยกต่างหากเหมือน `TEST_F` ของ Google Test — โค้ดก่อนหน้า `SECTION` แรก **รันซ้ำใหม่ทุกครั้ง**
สำหรับแต่ละ `SECTION` (คล้ายกับพฤติกรรมของ `SetUp()` แต่เขียนในที่เดียวกับ Test Case เลย):

```cpp
#define CATCH_CONFIG_MAIN
#include <catch2/catch_all.hpp>
#include <vector>
#include "sort_utils.hpp"

TEST_CASE("bubble_sort เรียงข้อมูลได้ถูกต้อง", "[bubble_sort]") {
    SECTION("ข้อมูลไม่เรียงลำดับทั่วไป") {
        std::vector<int> v = {8, 3, 5, 4, 9, 1};
        sortutils::bubble_sort(v);
        REQUIRE(v == std::vector<int>{1, 3, 4, 5, 8, 9});
    }

    SECTION("vector ว่างเปล่า") {
        std::vector<int> v;
        sortutils::bubble_sort(v);
        REQUIRE(v.empty());
    }

    SECTION("มีสมาชิกซ้ำกัน") {
        std::vector<int> v = {5, 3, 5, 1, 3};
        sortutils::bubble_sort(v);
        REQUIRE(v == std::vector<int>{1, 3, 3, 5, 5});
    }
}

TEST_CASE("quick_sort เรียงข้อมูลได้ถูกต้อง", "[quick_sort]") {
    SECTION("ข้อมูลไม่เรียงลำดับทั่วไป") {
        std::vector<int> v = {8, 3, 5, 4, 9, 1};
        sortutils::quick_sort(v);
        REQUIRE(v == std::vector<int>{1, 3, 4, 5, 8, 9});
    }

    SECTION("ข้อมูลเรียงกลับด้าน (worst case ของ quick sort ปกติ)") {
        std::vector<int> v = {5, 4, 3, 2, 1};
        sortutils::quick_sort(v);
        REQUIRE(v == std::vector<int>{1, 2, 3, 4, 5});
    }
}
```

```bash
g++ -Wall -Wextra -Wpedantic -std=c++17 test_sort_utils_catch2.cpp sort_utils.cpp \
    -o test_sort_utils_catch2 $(pkg-config --cflags --libs catch2-with-main)
./test_sort_utils_catch2
```

ผลลัพธ์จริง:

```
Randomness seeded to: 2726540501
===============================================================================
All tests passed (5 assertions in 2 test cases)
```

ลองรันแบบ verbose (`--success`) เพื่อดูว่าแต่ละ `SECTION` ถูกรันแยกกันจริงพร้อมรายงานผลทีละ
`REQUIRE`:

```bash
./test_sort_utils_catch2 --success
```

ผลลัพธ์จริง (แสดงบางส่วน):

```
-------------------------------------------------------------------------------
bubble_sort เรียงข้อมูลได้ถูกต้อง
  ข้อมูลไม่เรียงลำดับทั่วไป
-------------------------------------------------------------------------------
test_sort_utils_catch2.cpp:7
...............................................................................

test_sort_utils_catch2.cpp:10: PASSED:
  REQUIRE( v == std::vector<int>{1, 3, 4, 5, 8, 9} )
with expansion:
  { 1, 3, 4, 5, 8, 9 }
  ==
  { 1, 3, 4, 5, 8, 9 }
```

สังเกตว่า Catch2 พิมพ์ **ทั้งค่าที่คำนวณได้จริง (`with expansion:`) และค่าที่คาดหวัง** ออกมาให้
ดูโดยอัตโนมัติ แม้จะเขียนแค่ `REQUIRE(v == ...)` ธรรมดา (ไม่ต้องเขียน `REQUIRE_EQ` แยกแบบที่
Google Test ต้องใช้ `EXPECT_EQ` เพื่อให้ error message มีรายละเอียด) — นี่คือหนึ่งในจุดเด่นที่
ทำให้ Catch2 ได้รับความนิยมเรื่องความสะดวกในการอ่านผลลัพธ์เมื่อ test ล้มเหลว

### เชื่อมกับ CMake ด้วย find_package(Catch2)

```cmake
find_package(Catch2 3 REQUIRED)

add_executable(sort_utils_catch2 test_sort_utils_catch2.cpp)
target_link_libraries(sort_utils_catch2 PRIVATE sort_utils Catch2::Catch2WithMain)

include(CTest)
include(Catch)
catch_discover_tests(sort_utils_catch2)
```

Configure และ Build จริง (ต่อจากโปรเจกต์เดียวกับหัวข้อ 93.6 ที่มี Google Test อยู่แล้ว — สังเกต
ว่าโปรเจกต์เดียวใช้ **ทั้งสองเครื่องมือพร้อมกันได้** ไม่มีปัญหาอะไร):

```bash
cmake ..
```

```
-- Found GTest: /usr/lib/x86_64-linux-gnu/cmake/GTest/GTestConfig.cmake (found version "1.14.0")
-- Configuring done (0.4s)
-- Generating done (0.0s)
```

```bash
cmake --build .
./sort_utils_catch2
```

```
[ 83%] Building CXX object CMakeFiles/sort_utils_catch2.dir/test_sort_utils_catch2.cpp.o
[100%] Linking CXX executable sort_utils_catch2
[100%] Built target sort_utils_catch2
Randomness seeded to: 642185387
===============================================================================
All tests passed (5 assertions in 2 test cases)
```

`catch_discover_tests` ทำหน้าที่เดียวกับ `gtest_discover_tests` ของ Google Test — ลงทะเบียน
แต่ละ `TEST_CASE` เป็น CTest Test แยกกัน ทำให้รันผ่าน `ctest` ได้เหมือนกัน

---

## 93.8 เปรียบเทียบ Google Test กับ Catch2 (Step 743)

รัน `ctest` ในโปรเจกต์เดียวกันที่มีทั้ง Google Test และ Catch2 executable เพื่อดูว่าทั้งสอง
framework ทำงานร่วมกันภายใต้ระบบเดียวกันได้อย่างไร้รอยต่อ:

```bash
ctest
```

ผลลัพธ์จริง (ตัดมาบางส่วน แสดงให้เห็นว่า Test จากทั้งสอง framework ถูกนับรวมกันเป็นชุดเดียว):

```
      Start  1: BubbleSort.SortsUnorderedVector
 1/13 Test  #1: BubbleSort.SortsUnorderedVector ................ Passed
      ...
      Start 12: bubble_sort เรียงข้อมูลได้ถูกต้อง
12/13 Test #12: bubble_sort เรียงข้อมูลได้ถูกต้อง ................ Passed
      Start 13: quick_sort เรียงข้อมูลได้ถูกต้อง
13/13 Test #13: quick_sort เรียงข้อมูลได้ถูกต้อง .................. Passed

100% tests passed, 0 tests failed out of 13

Total Test time (real) =   0.04 sec
```

`ctest` ไม่สนใจเลยว่า Test แต่ละตัวมาจาก Google Test หรือ Catch2 — มันมองเห็นแค่ "executable
ที่ถูกลงทะเบียนไว้และคืนค่า exit code 0 เมื่อผ่าน" เท่านั้น (เชื่อมโยงกับความรู้เรื่อง Exit Code
จาก Part 1) นี่คือเหตุผลที่ทีมบางทีมเลือกใช้ทั้งสอง framework ผสมกันในโปรเจกต์เดียวได้โดยไม่มี
ปัญหาทางเทคนิคใดๆ (แม้จะไม่ใช่แนวทางที่แนะนำสำหรับโปรเจกต์ใหม่ก็ตาม เพราะเพิ่มความซับซ้อน
โดยไม่จำเป็น)

### ตารางเปรียบเทียบ

| ประเด็น | Google Test | Catch2 |
|---|---|---|
| **Syntax การนิยาม Test** | `TEST(Suite, Name)` | `TEST_CASE("คำอธิบาย", "[tag]")` |
| **Assertion หยุดทันที** | `ASSERT_*` | `REQUIRE(...)` |
| **Assertion ทำงานต่อ** | `EXPECT_*` | `CHECK(...)` |
| **แบ่ง sub-test ที่ใช้ setup ร่วมกัน** | ต้องเขียน `class` แยกด้วย `TEST_F` | เขียน `SECTION` ซ้อนใน `TEST_CASE` เดียว กระชับกว่า |
| **ข้อความ error เมื่อ assertion ล้มเหลว** | ต้องใช้ `EXPECT_EQ` (ไม่ใช่ `EXPECT_TRUE(a==b)`) เพื่อให้เห็นค่าทั้งสองฝั่ง | `REQUIRE(a == b)` ธรรมดาก็แสดงค่าทั้งสองฝั่งให้อัตโนมัติ |
| **Mocking Framework คู่กัน** | มี Google Mock (gMock) แยกต่างหาก ครบเครื่องมาก | ไม่มีในตัว ต้องพึ่ง Library อื่นเสริม (เช่น trompeloeil) |
| **Death Test** (ทดสอบว่าโปรแกรม crash/exit ตามที่คาดไว้) | มีในตัว (`EXPECT_DEATH`) | ไม่มีในตัวโดยตรง |
| **ขนาด/Compile Time** | ใหญ่กว่าเล็กน้อย ต้อง link เป็น library แยก | Catch2 3.x เป็น library ที่ compile แยกได้เหมือนกัน (รุ่นเก่ากว่านี้เป็น header-only ทำให้ compile time ต่อไฟล์นานกว่าถ้าใช้หลายไฟล์) |
| **ความนิยมในองค์กรใหญ่** | สูงมาก (Google เอง, และอีกหลายบริษัทเทคโนโลยีขนาดใหญ่) | สูงมากในสาย Open Source/Library ขนาดกลาง-เล็ก |
| **การอ่านค่า assertion ที่ล้มเหลว** | ชัดเจน แต่ต้องเลือก macro ให้ตรง (`EXPECT_EQ` ไม่ใช่ `EXPECT_TRUE`) | เป็นธรรมชาติกว่า เพราะแค่เขียนนิพจน์ตรงๆ ใน `REQUIRE` |

### เมื่อไหร่ควรเลือกอะไร

- เลือก **Google Test** เมื่อโปรเจกต์ต้องการ Mocking ที่ครบเครื่อง (Google Mock), ต้องการ
  Death Test, หรือทีมมีมาตรฐานเดียวกับระบบนิเวศของ Google อยู่แล้ว (เช่นใช้ Bazel เป็น Build
  System, ใช้ Abseil เป็น Library พื้นฐาน) — เหมาะกับโปรเจกต์ขนาดใหญ่ระดับองค์กร
- เลือก **Catch2** เมื่อต้องการ syntax ที่กระชับ อ่านง่าย เริ่มเขียน Test ได้เร็วโดยไม่ต้อง
  เรียนรู้ Assertion Macro เยอะ และไม่ต้องการ Mocking Framework ที่ซับซ้อน — เหมาะกับ Library
  ขนาดกลาง-เล็กหรือโปรเจกต์ Open Source ที่ต้องการลด Barrier ในการมีส่วนร่วม (Contribute)
  ของนักพัฒนาภายนอก
- ในทางปฏิบัติ **ทั้งสองตัวมีความสามารถเพียงพอสำหรับงานส่วนใหญ่** ความต่างที่แท้จริงมักขึ้นกับ
  รสนิยมของทีมและระบบนิเวศที่ใช้อยู่แล้วมากกว่าข้อจำกัดทางเทคนิคที่เป็นตัวชี้ขาด

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **เขียน Test Case ที่ไม่ได้เรียก Assertion ที่มีความหมายเลย (Always-True Test)** — นี่คือ
   กับดักที่อันตรายที่สุดของ Unit Testing เพราะ Test จะ "ผ่าน" เสมอไม่ว่าฟังก์ชันจะพังแค่ไหนก็ตาม
   ลองพิสูจน์ด้วยฟังก์ชันที่มีบั๊กจริง (`buggy_bubble_sort` ที่วนลูปผิดขอบเขตหนึ่งตำแหน่ง):
   ```cpp
   TEST(BadTestExample, AlwaysTruePitfall) {
       std::vector<int> v = {3, 1, 2};
       buggy_bubble_sort(v);
       // ลืมเรียก assertion ใดๆ ที่ตรวจผลลัพธ์จริง
       SUCCEED();
   }
   ```
   รันจริงได้ผลลัพธ์:
   ```
   [ RUN      ] BadTestExample.AlwaysTruePitfall
   [       OK ] BadTestExample.AlwaysTruePitfall (0 ms)
   ```
   Test "ผ่าน" ทั้งที่ `buggy_bubble_sort` เรียงข้อมูลผิดจริงๆ (ยืนยันได้จาก Test Case อื่นในไฟล์
   เดียวกันที่มี assertion จริงและ**ล้มเหลว**ด้วยฟังก์ชันเดียวกันนี้พอดี) — บทเรียนสำคัญ: ทุก
   Test Case ต้องมี Assertion อย่างน้อยหนึ่งตัวที่ตรวจสอบผลลัพธ์จริงเสมอ ห้ามเขียน Test เปล่าๆ
   ที่ผ่านโดยไม่มีความหมายใดๆ
2. **ใช้ `EXPECT_*` ในจุดที่ต้องใช้ `ASSERT_*`** — โดยเฉพาะก่อน dereference pointer หรือเข้าถึง
   index ของ container ทำให้โปรแกรม test ทั้งกระบวนการ Crash แทนที่จะรายงานผลว่า Test Case
   นั้นล้มเหลวอย่างสวยงาม (ดูตัวอย่างจริงในหัวข้อ 93.3)
3. **ลืม link `-lpthread`** — ทั้ง Google Test และ Catch2 (ผ่าน dependency บางส่วน) อาจ
   ต้องการ threading support ภายใน ถ้าลืม link จะได้ linker error `undefined reference to
   pthread_create` ทันที (เหมือนที่เจอกับ Google Benchmark ใน Part 90)
4. **Test Case พึ่งพาลำดับการรันของ Test Case อื่น** — เช่น Test A ตั้งค่า global variable ไว้
   แล้ว Test B คาดหวังว่า global variable นั้นจะมีค่าที่ Test A ตั้งไว้ ถ้า Framework รัน Test
   ตามลำดับอื่น (หรือรันแบบขนาน) ผลลัพธ์จะไม่แน่นอน ควรออกแบบทุก Test Case ให้เป็นอิสระจาก
   กันอย่างสมบูรณ์เสมอ (ใช้ Fixture's `SetUp()` เพื่อรีเซ็ตสถานะใหม่ทุกครั้งแทนการพึ่งพา state
   ที่หลงเหลือจาก Test อื่น)
5. **ทดสอบ Implementation Detail แทนที่จะทดสอบ Behavior** — เช่น เขียน Test ที่ตรวจสอบว่า
   ฟังก์ชัน sort เรียก `std::swap` กี่ครั้ง (รายละเอียดภายในที่เปลี่ยนได้ตลอดเวลาตอน Refactor)
   แทนที่จะตรวจแค่ว่า "ผลลัพธ์สุดท้ายเรียงถูกต้องหรือไม่" (พฤติกรรมภายนอกที่ผู้ใช้จริงสนใจ) การ
   ทดสอบแบบแรกทำให้ Test พังทุกครั้งที่ Refactor แม้พฤติกรรมจริงจะยังถูกต้องอยู่ ขัดกับเป้าหมาย
   หลักของ Unit Test ที่ควรทำให้ Refactor ได้อย่างมั่นใจมากขึ้น ไม่ใช่น้อยลง
6. **ไม่ครอบคลุม Edge Case** — เขียนแค่ Test Case สำหรับข้อมูลปกติ (Happy Path) แต่ลืม
   ทดสอบกรณีขอบ เช่น container ว่างเปล่า, สมาชิกตัวเดียว, ข้อมูลซ้ำกันทั้งหมด, หรือข้อมูลที่
   เรียงอยู่แล้ว (ดูตัวอย่างการครอบคลุม Edge Case อย่างครบถ้วนในหัวข้อ 93.6) บั๊กที่พบบ่อยที่สุด
   ในโปรแกรมจริงมักซ่อนอยู่ที่ Edge Case เหล่านี้ ไม่ใช่ในกรณีปกติทั่วไป

---

## แบบฝึกหัดท้ายบท

1. เขียน Unit Test ด้วย Google Test สำหรับฟังก์ชัน `mu_is_prime` จาก Part 17/18 (พอร์ตเป็น
   C++ ก่อนถ้าจำเป็น) ครอบคลุมอย่างน้อย: จำนวนเฉพาะจริง (2, 17), จำนวนไม่เฉพาะ (4, 100),
   ค่าติดลบ, และ 0 กับ 1 (Edge Case ที่นิยามว่าไม่ใช่จำนวนเฉพาะ)
2. เขียน Unit Test เดียวกันกับข้อ 1 ด้วย Catch2 แทน โดยใช้ `SECTION` แบ่งกรณีต่างๆ ให้เป็น
   ระเบียบ
3. ใช้ Google Test Fixture (`TEST_F`) เขียน Test สำหรับ Stack ที่เคยเรียนใน Part 20 (Array-
   based Stack) ให้ Fixture เตรียม Stack ที่มีข้อมูล 3 ตัวอยู่แล้วก่อนทุก Test Case ทดสอบทั้ง
   `push`, `pop`, และการ throw exception เมื่อ `pop` จาก Stack ว่าง
4. หยิบฟังก์ชัน `buggy_bubble_sort` จากหัวข้อ Common Pitfalls มาแก้บั๊กให้ถูกต้อง แล้วรัน Test
   Suite เดิมอีกครั้งเพื่อยืนยันว่า Test ที่เคยล้มเหลวกลับมาผ่านทั้งหมด
5. เขียน `CMakeLists.txt` ที่มี executable สองตัว: หนึ่งใช้ Google Test อีกตัวใช้ Catch2 ทดสอบ
   ฟังก์ชันชุดเดียวกัน (`sort_utils` จากหัวข้อ 93.5) แล้วรัน `ctest` ให้เห็นผลทั้งสองชุดรวมกัน
6. (โบนัส) ค้นคว้าเพิ่มเติมเรื่อง **Google Mock** (มาพร้อม Google Test) แล้วลองเขียนตัวอย่าง
   ง่ายๆ ที่ใช้ Mock Object แทน Interface สมมติ (เช่น `class Logger` ที่มี virtual function
   `void log(const std::string&)`) เพื่อทดสอบว่าฟังก์ชันหนึ่งเรียก `log()` ถูกจำนวนครั้งตามที่
   คาดหวังหรือไม่

### แนวทางเฉลยข้อ 1

```cpp
#include <gtest/gtest.h>

// พอร์ต mu_is_prime จาก Part 17/18 มาเป็นฟังก์ชัน C++ ธรรมดา (logic เหมือนต้นฉบับทุกประการ)
bool is_prime(int n) {
    if (n < 2) {
        return false;
    }
    for (int i = 2; static_cast<long>(i) * i <= n; i++) {
        if (n % i == 0) {
            return false;
        }
    }
    return true;
}

TEST(IsPrime, RecognizesPrimeNumbers) {
    EXPECT_TRUE(is_prime(2));
    EXPECT_TRUE(is_prime(17));
    EXPECT_TRUE(is_prime(97));
}

TEST(IsPrime, RecognizesNonPrimeNumbers) {
    EXPECT_FALSE(is_prime(4));
    EXPECT_FALSE(is_prime(100));
}

TEST(IsPrime, HandlesNegativeNumbers) {
    EXPECT_FALSE(is_prime(-7));
    EXPECT_FALSE(is_prime(-1));
}

TEST(IsPrime, HandlesZeroAndOneAsNotPrime) {
    // ตามนิยามคณิตศาสตร์ 0 และ 1 ไม่ใช่จำนวนเฉพาะ -- Edge Case ที่มักถูกลืมทดสอบ
    EXPECT_FALSE(is_prime(0));
    EXPECT_FALSE(is_prime(1));
}
```

ทดสอบจริง:

```bash
$ g++ -Wall -Wextra -Wpedantic -std=c++17 test_is_prime.cpp -o test_is_prime \
    -lgtest -lgtest_main -lpthread
$ ./test_is_prime
[==========] Running 4 tests from 1 test suite.
[----------] 4 tests from IsPrime
[ RUN      ] IsPrime.RecognizesPrimeNumbers
[       OK ] IsPrime.RecognizesPrimeNumbers (0 ms)
[ RUN      ] IsPrime.RecognizesNonPrimeNumbers
[       OK ] IsPrime.RecognizesNonPrimeNumbers (0 ms)
[ RUN      ] IsPrime.HandlesNegativeNumbers
[       OK ] IsPrime.HandlesNegativeNumbers (0 ms)
[ RUN      ] IsPrime.HandlesZeroAndOneAsNotPrime
[       OK ] IsPrime.HandlesZeroAndOneAsNotPrime (0 ms)
[==========] 4 tests from 1 test suite ran. (0 ms total)
[  PASSED  ] 4 tests.
```

### แนวทางเฉลยข้อ 4

บั๊กเดิมของ `buggy_bubble_sort` คือขอบเขตของลูปชั้นในผิดไปหนึ่งตำแหน่ง:

```cpp
// เวอร์ชันมีบั๊ก (จากหัวข้อ Common Pitfalls)
inline void buggy_bubble_sort(std::vector<int>& v) {
    const std::size_t n = v.size();
    if (n < 2) return;
    for (std::size_t i = 0; i < n - 1; i++) {
        for (std::size_t j = 0; j + 2 < n - i; j++) {  // บั๊ก: ควรเป็น j + 1 < n - i
            if (v[j] > v[j + 1]) {
                std::swap(v[j], v[j + 1]);
            }
        }
    }
}
```

แก้บั๊กโดยเปลี่ยนเงื่อนไขของลูปชั้นในให้ถูกต้อง:

```cpp
// เวอร์ชันแก้ไขแล้ว
inline void fixed_bubble_sort(std::vector<int>& v) {
    const std::size_t n = v.size();
    if (n < 2) return;
    for (std::size_t i = 0; i < n - 1; i++) {
        for (std::size_t j = 0; j + 1 < n - i; j++) {  // แก้แล้ว: j + 1 < n - i
            if (v[j] > v[j + 1]) {
                std::swap(v[j], v[j + 1]);
            }
        }
    }
}
```

รัน Test Suite เดิม (`BuggyBubbleSort.SortsUnorderedVector` ที่เคยล้มเหลว) อีกครั้งด้วยฟังก์ชัน
ที่แก้ไขแล้ว:

```bash
$ g++ -Wall -Wextra -Wpedantic -std=c++17 test_fixed.cpp -o test_fixed \
    -lgtest -lgtest_main -lpthread
$ ./test_fixed
[ RUN      ] FixedBubbleSort.SortsUnorderedVector
[       OK ] FixedBubbleSort.SortsUnorderedVector (0 ms)
[==========] 1 test from 1 test suite ran. (0 ms total)
[  PASSED  ] 1 test.
```

Test ที่เคยล้มเหลว (`Expected equality of these values: v ... Which is: { 3, 4, 5, 8, 9, 1 }`
ตามที่เห็นในหัวข้อ Common Pitfalls) กลับมาผ่านทันทีหลังแก้ไขขอบเขตของลูป — นี่คือตัวอย่างที่
เป็นรูปธรรมที่สุดของประโยชน์ของ Unit Test ตามที่อธิบายไว้ในหัวข้อ 93.1: Test ทำหน้าที่เป็น
"หลักฐานที่วัดผลได้" ว่าการแก้ไขโค้ดนั้นถูกต้องจริง ไม่ใช่แค่ความรู้สึกของผู้เขียนโค้ดว่า "น่าจะถูก
แล้ว"

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่าทำไม Unit Test ถึงจำเป็นสำหรับซอฟต์แวร์ระดับมืออาชีพ (ป้องกัน Regression, ทำให้
  Refactor ได้อย่างมั่นใจ) และหลักการ Arrange-Act-Assert ที่เป็นโครงร่างของ Test Case ที่ดี
- เขียน Test Case แรกด้วย Google Test ผ่าน `TEST()` และพิสูจน์ด้วยการรันจริงว่า `ASSERT_*`
  หยุด Test Case ทันทีเมื่อล้มเหลว ในขณะที่ `EXPECT_*` ทำงานต่อไปได้
- เขียน Test Fixture ด้วย `TEST_F` เพื่อใช้ข้อมูลตั้งต้นร่วมกันระหว่างหลาย Test Case โดยที่แต่ละ
  Test Case ยังคงเป็นอิสระจากกันอย่างสมบูรณ์
- Build โปรเจกต์ที่ใช้ Google Test ผ่าน CMake ได้สำเร็จด้วย `find_package(GTest)` และเข้าใจ
  ข้อจำกัดของ `FetchContent` เมื่อ network ถูกจำกัด (พิสูจน์ด้วย error จริงที่เกิดขึ้นในบทเรียนนี้)
- เขียน Test Case ด้วย Catch2 ผ่าน `TEST_CASE`, `REQUIRE`, และ `SECTION` พร้อมเห็นว่า
  Catch2 แสดงค่าที่คำนวณได้จริงเทียบกับค่าคาดหวังให้อัตโนมัติเมื่อ assertion ล้มเหลว
- เปรียบเทียบ Google Test กับ Catch2 อย่างละเอียด และรู้ว่าทั้งสองทำงานร่วมกันภายใต้ `ctest`
  ได้อย่างไร้รอยต่อ
- เขียนชุด Unit Test ที่สมบูรณ์ให้กับ `bubble_sort`/`quick_sort` ที่พอร์ตมาจาก Part 23 ครอบคลุม
  ทั้ง Happy Path และ Edge Case สำคัญทั้งหมด (ว่างเปล่า, สมาชิกเดียว, ข้อมูลซ้ำ, เรียงอยู่แล้ว,
  เรียงกลับด้าน) และพิสูจน์ประโยชน์ของ Unit Test ด้วยการแก้บั๊กจริงแล้วดู Test เปลี่ยนจากล้มเหลว
  เป็นผ่าน

ตอนนี้เรามีเครื่องมือครบสำหรับสร้างโครงสร้างโปรเจกต์ (CMake), จัดการ Dependency ภายนอก
(vcpkg/Conan), และตรวจสอบความถูกต้องของโค้ดอัตโนมัติ (Google Test/Catch2) แล้ว ใน
**Part 94** เราจะนำทุกอย่างมาผูกเข้ากับ **Continuous Integration (CI)** ด้วย **GitHub
Actions** เพื่อให้ทุกครั้งที่ push โค้ดเข้า repository ระบบ build และรัน Test ทั้งหมดให้อัตโนมัติ
โดยไม่ต้องรันด้วยมือเองอีกต่อไป — เป็นก้าวสำคัญสู่การทำงานแบบทีมวิศวกรรมมืออาชีพเต็มรูปแบบ

**ต่อไป:** Part 94 — Continuous Integration สำหรับ C++ ด้วย GitHub Actions
