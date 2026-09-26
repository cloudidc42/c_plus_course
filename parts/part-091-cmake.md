# Part 91: CMake ตั้งแต่พื้นฐานถึงขั้นสูง (Step 721–728)

> Module H — Build Systems, Testing และ Tooling | Part 91 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 721–728
> Part ก่อนหน้า: [Part 90 — Benchmarking ด้วย Google Benchmark](./part-090-google-benchmark.md) | Part ถัดไป: [Part 92 — Package Management: Conan และ vcpkg](./part-092-package-management.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไม Makefile ที่เขียนตรงๆ (จาก Part 18) ถึงมีข้อจำกัดเรื่อง Portability เมื่อ
   โปรเจกต์ต้อง build ข้าม Platform/Compiler/IDE และ CMake แก้ปัญหานี้ในฐานะ
   "meta-build-system" ได้อย่างไร
2. เขียน `CMakeLists.txt` พื้นฐานด้วย `cmake_minimum_required`, `project`, `add_executable`
   และ configure/build โปรเจกต์แบบ **out-of-source build** ได้อย่างถูกต้อง
3. อธิบายและใช้แนวทาง **Target-based** สมัยใหม่ (`target_include_directories`,
   `target_link_libraries`, `target_compile_features`) แทนตัวแปร global แบบเก่าที่ CMake
   รุ่นก่อนหน้าใช้ (`include_directories`, `link_libraries`, `CMAKE_CXX_FLAGS`)
4. สร้างและใช้งาน Library ด้วย `add_library` แล้ว link เข้ากับ Executable ผ่าน
   `target_link_libraries` พร้อมเข้าใจความหมายของ `PUBLIC`/`PRIVATE`/`INTERFACE`
5. จัดการโปรเจกต์ที่มีหลายโฟลเดอร์ย่อยด้วย `add_subdirectory` เพื่อให้โปรเจกต์ขนาดใหญ่
   เป็นระเบียบและ scale ได้
6. ใช้ `CMAKE_BUILD_TYPE` สลับระหว่างโหมด Debug และ Release ได้อย่างถูกต้อง และเข้าใจว่า
   flag ของ compiler เปลี่ยนไปอย่างไรจริงๆ ในแต่ละโหมด
7. Port โปรเจกต์ `math_utils` จาก Part 18 (ที่เดิมใช้ Makefile) มาใช้ CMake แทนได้อย่างสมบูรณ์
   พร้อมพิสูจน์ว่า CMake ติดตาม dependency ของ header ให้อัตโนมัติโดยไม่ต้องเขียน
   `-MMD -MP` เองเหมือนที่ทำใน Makefile

---

## 91.1 ทำไมต้องมี CMake (Step 721)

ใน **Part 18** เราเขียน Makefile จนจบระดับที่ scale ได้กับโปรเจกต์ C จำนวนไฟล์เยอะๆ ด้วย
`wildcard`, `patsubst`, และ auto-dependency generation (`-MMD -MP`) ซึ่งเพียงพอมากสำหรับ
โปรเจกต์ที่ compile บนเครื่อง Linux เครื่องเดียว ด้วย compiler ตัวเดียว (`gcc`) เสมอ

แต่พอโปรเจกต์ C++ โตขึ้นถึงระดับที่ใช้งานจริงในอุตสาหกรรม ข้อจำกัดของ Makefile ที่เขียนตรงๆ
เริ่มปรากฏชัดขึ้นเรื่อยๆ:

1. **ไม่ Portable ข้าม Platform**: Makefile ที่เขียนใน Part 18 สมมติว่ามี `gcc`, ใช้ syntax ของ
   `rm -f` (คำสั่ง Unix) และ path แบบ `/` เสมอ พอย้ายไปรันบน Windows (ที่ไม่มี `make` มาให้
   ในตัวและ shell คนละแบบ) หรือแม้แต่ macOS ที่บาง keyword ของ shell ต่างจาก Linux เล็กน้อย
   Makefile เดิมอาจพังทันที
2. **ไม่รู้จัก IDE/Generator อื่น**: นักพัฒนาจำนวนมากใช้ Visual Studio บน Windows หรือ Xcode
   บน macOS ซึ่งมีระบบ Project File ของตัวเอง (`.sln`, `.xcodeproj`) ไม่ใช่ Makefile — ถ้าอยาก
   ให้โปรเจกต์เดียวกันเปิดได้ทั้ง Visual Studio, Xcode และ command line บน Linux พร้อมกัน
   การเขียน Makefile มือเปล่าไม่มีทางทำได้เลย ต้องดูแลไฟล์ config แยกกันหลายชุด
3. **จัดการ Dependency ข้ามระบบยาก**: การหาว่า library ตัวหนึ่ง (เช่น OpenSSL, Boost) ติดตั้งอยู่
   ที่ path ไหนบนเครื่องของแต่ละคน ต่างกันไปตาม distro/OS การเขียน path แบบ hardcode ใน
   Makefile ใช้ได้แค่บนเครื่องของผู้เขียนเท่านั้น
4. **ไม่มีกลไกมาตรฐานสำหรับ Configuration ที่ซับซ้อน**: การตรวจสอบว่า compiler รองรับ feature
   ใดบ้าง (เช่น C++20 Concepts) หรือ library ตัวไหนมีอยู่บนเครื่องหรือไม่ ต้องเขียน shell script
   ตรวจสอบเองทั้งหมด ซึ่งเสี่ยงบั๊กสูงและไม่มีมาตรฐานร่วมกัน

**CMake** คือคำตอบของอุตสาหกรรมสำหรับปัญหานี้ทั้งหมด แต่มีจุดที่ต้องเข้าใจให้ถูกต้องตั้งแต่แรก:

> **CMake ไม่ใช่ Build System — CMake คือ "Meta-Build-System" (หรือ Build System Generator)**

CMake **ไม่ได้ compile โค้ดเอง** แต่มันอ่านไฟล์ `CMakeLists.txt` ที่เราเขียน (ซึ่งอธิบายโปรเจกต์
ในระดับสูง เช่น "มี executable ชื่อนี้ ประกอบจากไฟล์เหล่านี้ ต้อง link กับ library เหล่านี้") แล้ว
**generate** ไฟล์ build ที่แท้จริงให้เหมาะกับแต่ละแพลตฟอร์มโดยอัตโนมัติ:

```
                    CMakeLists.txt (เขียนครั้งเดียว อธิบายโปรเจกต์แบบ abstract)
                            │
                            ▼
                    ┌───────────────┐
                    │  cmake (ทำหน้าที่ Generator) │
                    └───────────────┘
                            │
        ┌───────────────────┼───────────────────┬─────────────────────┐
        ▼                   ▼                   ▼                      ▼
   Unix Makefiles        Ninja build          Visual Studio .sln    Xcode .xcodeproj
   (Linux/macOS)      (เร็วกว่า Make มาก)      (Windows)              (macOS)
```

พูดง่ายๆ คือ CMake เป็นเหมือน "นักแปล" ที่แปลงคำอธิบายโปรเจกต์ภาษาเดียวที่เราเขียน ให้กลาย
เป็นไฟล์ build ที่ tool เฉพาะของแต่ละแพลตฟอร์มเข้าใจ — เราเขียน `CMakeLists.txt` แค่ไฟล์เดียว
แล้วให้ CMake ไป generate Makefile บน Linux, Ninja build file บนเครื่องที่ติดตั้ง Ninja, หรือ
Visual Studio Solution บน Windows ให้เองโดยอัตโนมัติ

### พิสูจน์แนวคิดนี้ด้วยการรันจริง

เพื่อให้เห็นภาพชัดว่า CMake เป็น "ตัวสร้าง" ไม่ใช่ "ตัว build" เอง เรามาลองสร้างโปรเจกต์เล็กๆ
แล้วสั่งให้ CMake generate ไฟล์ build **สองแบบที่ต่างกัน** จาก `CMakeLists.txt` ไฟล์เดียวกัน

ก่อนอื่นติดตั้ง CMake (ถ้ายังไม่มี) และ Ninja (Generator ทางเลือกที่เร็วกว่า Make):

```bash
sudo apt update
sudo apt install cmake ninja-build -y
cmake --version
```

บนเครื่องที่ใช้เขียนบทเรียนนี้ได้เวอร์ชัน:

```
$ cmake --version
cmake version 3.28.3

CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

สร้างโปรเจกต์เล็กที่สุดเท่าที่จะเป็นไปได้ — ไฟล์ `main.cpp`:

```cpp
#include <cstdio>

int main() {
    printf("Hello, CMake!\n");
    return 0;
}
```

และไฟล์ `CMakeLists.txt` (จะอธิบายทุก keyword อย่างละเอียดในหัวข้อ 91.2 ถัดไป):

```cmake
cmake_minimum_required(VERSION 3.20)
project(HelloCMake LANGUAGES CXX)

add_executable(hello main.cpp)
```

**ครั้งที่ 1: ให้ CMake generate Makefile (ค่า default บน Linux)**

```bash
mkdir build_make && cd build_make
cmake .. -G "Unix Makefiles"
ls
```

```
CMakeCache.txt  CMakeFiles  Makefile  cmake_install.cmake
```

สังเกตว่ามีไฟล์ `Makefile` เกิดขึ้นจริง! นี่คือ Makefile ที่ CMake generate ให้อัตโนมัติ (ซับซ้อน
กว่า Makefile ที่เขียนมือใน Part 18 มาก เพราะต้องรองรับ feature และ edge case สารพัด) เรา
สามารถสั่ง `make` ต่อได้ตามปกติ

**ครั้งที่ 2: ให้ CMake generate ไฟล์ของ Ninja แทน จาก `CMakeLists.txt` ไฟล์เดิมทุกตัวอักษร**

```bash
cd .. && mkdir build_ninja && cd build_ninja
cmake .. -G Ninja
ls
```

ผลลัพธ์จริง:

```
CMakeCache.txt  CMakeFiles  build.ninja  cmake_install.cmake
```

```bash
cmake --build .
./hello
```

ผลลัพธ์จริง:

```
[1/2] Building CXX object CMakeFiles/hello.dir/main.cpp.o
[2/2] Linking CXX executable hello
Hello, CMake!
```

**นี่คือหัวใจสำคัญที่สุดของ Part นี้**: เราเขียน `CMakeLists.txt` เพียงไฟล์เดียว ไม่ได้แก้ไขอะไร
เลยระหว่างสองครั้ง แต่ CMake generate ไฟล์ build คนละระบบไปให้โดยสมบูรณ์ (`Makefile` กับ
`build.ninja`) — ถ้าเป็น Windows ก็สามารถสั่ง `cmake .. -G "Visual Studio 17 2022"` เพื่อ
generate ไฟล์ `.sln` ของ Visual Studio จาก `CMakeLists.txt` ไฟล์เดียวกันนี้ได้เช่นกัน โดยไม่ต้อง
แก้ไขอะไรเลยแม้แต่บรรทัดเดียว

ยิ่งไปกว่านั้น เราไม่ได้ผูกกับ compiler ตัวใดตัวหนึ่งด้วย ลองสั่งให้ CMake ใช้ Clang แทน GCC
จาก `CMakeLists.txt` เดิมทุกประการ:

```bash
cd .. && mkdir build_clang && cd build_clang
cmake .. -DCMAKE_CXX_COMPILER=clang++
```

ผลลัพธ์จริง (เห็นบรรทัด compiler identification เปลี่ยนจาก GNU เป็น Clang):

```
-- The CXX compiler identification is Clang 18.1.3
-- Configuring done (0.4s)
```

```bash
cmake --build .
./hello
```

```
[ 50%] Building CXX object CMakeFiles/hello.dir/main.cpp.o
[100%] Linking CXX executable hello
Hello, CMake!
```

นี่คือความหมายที่แท้จริงของคำว่า **Portable**: `CMakeLists.txt` ไฟล์เดียวใช้ได้กับทั้ง 2
Generator (Make, Ninja) และ 2 Compiler (GCC, Clang) โดยไม่ต้องแก้ไขโค้ดแม้แต่บรรทัดเดียว
ซึ่งเป็นสิ่งที่ Makefile ที่เขียนมือแบบ Part 18 ทำไม่ได้เลยถ้าไม่เขียน logic เพิ่มเติมเอง

> **หมายเหตุสำคัญ**: จาก Part นี้เป็นต้นไป ตัวอย่างส่วนใหญ่จะใช้ค่า default ของ Generator
> (Unix Makefiles บน Linux) เพื่อความเรียบง่าย เพราะสิ่งที่เรียนรู้ทั้งหมดเกี่ยวกับ `CMakeLists.txt`
> ใช้ได้เหมือนกันทุกประการไม่ว่าจะเลือก Generator ใด

---

## 91.2 CMakeLists.txt พื้นฐานและ Out-of-Source Build (Step 722)

### สามคำสั่งที่ต้องมีในทุกโปรเจกต์

```cmake
cmake_minimum_required(VERSION 3.20)
project(HelloCMake LANGUAGES CXX)

add_executable(hello main.cpp)
```

| คำสั่ง | ความหมาย |
|---|---|
| `cmake_minimum_required(VERSION x.y)` | ประกาศเวอร์ชัน CMake ต่ำสุดที่ไฟล์นี้ต้องการ **ต้องเป็นบรรทัดแรกสุดเสมอ** เพราะ CMake ใช้เวอร์ชันนี้กำหนดพฤติกรรม (Policy) ของคำสั่งอื่นๆ ทั้งหมดที่ตามมา |
| `project(ชื่อ LANGUAGES ...)` | ตั้งชื่อโปรเจกต์และประกาศภาษาที่ใช้ (`CXX` = C++, `C` = C) CMake จะใช้ข้อมูลนี้ตรวจสอบหา compiler ที่เหมาะสมให้อัตโนมัติ |
| `add_executable(ชื่อ ไฟล์ต้นฉบับ...)` | สร้าง **target** ที่เป็นโปรแกรม executable จากไฟล์ source ที่ระบุ |

คำว่า **target** เป็นคำศัพท์สำคัญที่สุดใน CMake สมัยใหม่ (จะพูดถึงเจาะลึกในหัวขัด 91.4) —
ทุกอย่างที่ CMake "สร้าง" ไม่ว่าจะเป็นโปรแกรม หรือ Library ล้วนเรียกว่า target ทั้งสิ้น

### กระบวนการทำงานของ CMake มี 2 ขั้นตอนเสมอ

```
CMakeLists.txt
      │
      ▼
[1] Configure (cmake ..)   →  ตรวจสอบ compiler, สร้างไฟล์ build (Makefile/Ninja/...)
      │
      ▼
[2] Build (cmake --build .)  →  รันไฟล์ build จริงเพื่อ compile/link โปรแกรม
```

ขั้นตอน **Configure** คือตอนที่ CMake อ่าน `CMakeLists.txt`, ตรวจสอบว่าระบบมี compiler ที่
ใช้งานได้หรือไม่, ค้นหา library ที่ต้องการ (ถ้ามี), แล้ว **generate** ไฟล์ build ออกมา ส่วนขั้นตอน
**Build** คือตอนที่ระบบ build ที่ถูก generate ไว้ (เช่น `make` หรือ `ninja`) ทำงานจริงเพื่อ compile
และ link ไฟล์ทั้งหมด

### Out-of-Source Build คืออะไร และทำไมสำคัญมาก

> **กฎทองข้อแรกของ CMake**: ห้าม configure โปรเจกต์ในโฟลเดอร์เดียวกับ Source Code
> เด็ดขาด ให้สร้างโฟลเดอร์แยกต่างหาก (มักตั้งชื่อ `build`) แล้ว configure ในนั้นเสมอ

รูปแบบมาตรฐานที่ใช้กันทั่วโลก:

```bash
mkdir build
cd build
cmake ..
cmake --build .
```

มาดูผลลัพธ์จริงจากการทำตามขั้นตอนนี้กับโปรเจกต์ `HelloCMake` ด้านบน:

```bash
$ ls
CMakeLists.txt  main.cpp

$ mkdir build && cd build
$ cmake ..
```

ผลลัพธ์จริง:

```
-- The CXX compiler identification is GNU 13.3.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done (0.2s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/HelloCMake/build
```

```bash
$ cmake --build .
```

```
[ 50%] Building CXX object CMakeFiles/hello.dir/main.cpp.o
[100%] Linking CXX executable hello
[100%] Built target hello
```

```bash
$ ./hello
Hello, CMake!
```

หลังจากนี้เมื่อกลับไปดูที่โฟลเดอร์หลักของโปรเจกต์:

```bash
$ cd ..
$ ls
CMakeLists.txt  build  main.cpp
```

สังเกตว่า **โฟลเดอร์ source** (`CMakeLists.txt`, `main.cpp`) **สะอาดเหมือนเดิมทุกประการ** —
ไฟล์ที่เกิดจากการ build ทั้งหมด (object file, executable, CMakeCache.txt, Makefile ที่ CMake
generate ให้) ถูกเก็บอยู่ในโฟลเดอร์ `build/` เพียงโฟลเดอร์เดียว

ประโยชน์ของ out-of-source build มีอย่างน้อย 3 ข้อสำคัญ:

1. **ลบทิ้งได้ปลอดภัย 100%**: อยาก build ใหม่ทั้งหมด (clean build) แค่ `rm -rf build` แล้ว
   สร้างใหม่ — ไม่มีความเสี่ยงเผลอลบ source code เลย เพราะไฟล์ที่สร้างขึ้นทั้งหมดแยกออกจาก
   source code อย่างสมบูรณ์
2. **`.gitignore` ทำได้ง่ายมาก**: เพิ่ม `build/` บรรทัดเดียวใน `.gitignore` ก็ครอบคลุมไฟล์ที่เกิด
   จากการ build ทั้งหมด ไม่ต้องมานั่งไล่ ignore ไฟล์ `.o`, `.exe` ทีละไฟล์เหมือน Makefile
3. **Build ได้หลาย Configuration พร้อมกัน**: สร้าง `build_debug/` กับ `build_release/` แยกกัน
   จาก source code ชุดเดียวกันได้ ไม่ชนกัน (จะเห็นตัวอย่างจริงในหัวข้อ 91.7)

---

## 91.3 กับดักของ In-Source Build (Step 723)

เพื่อให้เห็นปัญหาของ in-source build อย่างเป็นรูปธรรม ลองทำผิดกฎทองข้างบนดูจริงๆ — สั่ง
`cmake .` ในโฟลเดอร์เดียวกับ `CMakeLists.txt` ตรงๆ (ไม่สร้างโฟลเดอร์ `build` แยก):

```bash
$ ls
CMakeLists.txt  main.cpp

$ cmake .
```

ผลลัพธ์จริง — CMake **ยอมทำให้จริงๆ** โดยไม่ error หรือเตือนอะไรเป็นพิเศษ:

```
-- The CXX compiler identification is GNU 13.3.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done (0.2s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/HelloCMake
```

```bash
$ cmake --build .
```

```
[ 50%] Building CXX object CMakeFiles/hello.dir/main.cpp.o
[100%] Linking CXX executable hello
[100%] Built target hello
```

ตอนนี้มาดูว่าโฟลเดอร์ source กลายเป็นอะไร:

```bash
$ ls -la
total 64
drwxr-xr-x 3 user user  4096 Sep 26 08:41 .
drwxr-xr-x 4 user user  4096 Sep 26 08:41 ..
-rw-r--r-- 1 user user 13008 Sep 26 08:41 CMakeCache.txt
drwxr-xr-x 6 user user  4096 Sep 26 08:41 CMakeFiles
-rw-r--r-- 1 user user   103 Sep 26 08:41 CMakeLists.txt
-rw-r--r-- 1 user user  5488 Sep 26 08:41 Makefile
-rw-r--r-- 1 user user  1786 Sep 26 08:41 cmake_install.cmake
-rwxr-xr-x 1 user user 15960 Sep 26 08:41 hello
-rw-r--r-- 1 user user    79 Sep 26 08:41 main.cpp
```

โฟลเดอร์ source ที่เดิมมีแค่ 2 ไฟล์ (`CMakeLists.txt`, `main.cpp`) ตอนนี้เต็มไปด้วยไฟล์และ
โฟลเดอร์ที่ CMake สร้างขึ้นปนกับ source code จริงถึง 6 รายการ! ปัญหาที่ตามมา:

- ถ้าเผลอสั่ง `git add .` จะได้ commit ไฟล์ขยะที่ generate ได้ใหม่เสมอเข้าไปใน repository
  (แถม `CMakeCache.txt` เก็บ absolute path ของเครื่องที่ configure ไว้ด้วย ถ้า commit ไปแล้ว
  คนอื่น pull มาใช้จะพังทันทีเพราะ path ไม่ตรงกับเครื่องเขา)
- อยาก clean build ใหม่ทั้งหมดต้องมานั่งลบไฟล์ทีละตัว (หรือเสี่ยงเขียน `rm -rf *` ผิดพลาดจน
  ลบ source code ทิ้งไปด้วย)
- ถ้าต้องการทดสอบ 2 configuration พร้อมกัน (เช่น Debug กับ Release) จะชนกันในโฟลเดอร์
  เดียวกันทันที

**วิธีแก้เมื่อพลาดไปแล้ว**: ลบไฟล์ที่ CMake สร้างทั้งหมดออก แล้วเริ่ม out-of-source build ใหม่

```bash
rm -rf CMakeCache.txt CMakeFiles Makefile cmake_install.cmake hello
mkdir build && cd build && cmake .. && cmake --build .
```

นี่คือเหตุผลที่ทุกบทเรียนและทุกโปรเจกต์ตัวอย่างใน Part นี้ (และ Part ถัดๆ ไปทั้งหมดของ
หลักสูตร) จะใช้รูปแบบ `mkdir build && cd build && cmake .. && cmake --build .` เป็นมาตรฐาน
ตายตัว — เป็นธรรมเนียมที่ทีมวิศวกรรมมืออาชีพทุกทีมยึดถือ

---

## 91.4 แนวทาง Target-based สมัยใหม่ (Step 724)

CMake รุ่นเก่า (ก่อน CMake 3.0 หรือที่เรียกว่า "Old-Style CMake") ใช้ตัวแปร **Global** ในการ
กำหนดค่าต่างๆ ซึ่งมีปัญหาสำคัญ: ค่าที่ตั้งจะ**มีผลกับทุก target ในไฟล์ที่ตามมาทั้งหมด**
ไม่สามารถควบคุมแบบละเอียดต่อ target ได้ ตัวอย่างสไตล์เก่าที่ไม่แนะนำให้เขียนอีกต่อไป:

```cmake
# ⚠️ Old-Style CMake — ไม่แนะนำให้ใช้แล้วในโปรเจกต์ใหม่
include_directories(include)              # มีผลกับ "ทุก target" ที่ประกาศหลังจากนี้
link_libraries(mathutils)                 # มีผลกับ "ทุก target" เช่นกัน
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")  # ตัวแปร Global แก้ไขได้จากทุกที่
```

ปัญหาคือถ้าโปรเจกต์มี 5 executable ที่แต่ละตัวต้องการ include path หรือ library ที่ต่างกัน
คำสั่งแบบ Global นี้ไม่สามารถแยกแยะได้เลยว่า "ตัวไหนต้องการอะไร" — ทุกอย่างถูกยัดรวมกันหมด
ทำให้เมื่อโปรเจกต์โตขึ้น การไล่ debug ว่าทำไม target หนึ่งถึงมองเห็น include path ของอีก target
กลายเป็นเรื่องปวดหัวมาก

**CMake สมัยใหม่ (Modern CMake, ตั้งแต่ CMake 3.x เป็นต้นไป)** แก้ปัญหานี้ด้วยแนวคิด
**Target-based**: ทุกการตั้งค่า (include path, library ที่ link, compile flag, C++ standard)
ผูกติดกับ **target ที่ระบุชื่อชัดเจน** เท่านั้น ไม่กระทบ target อื่นโดยไม่ได้ตั้งใจ

| คำสั่งแบบเก่า (Global) | คำสั่งแบบใหม่ (Target-based) | ผลกระทบ |
|---|---|---|
| `include_directories(...)` | `target_include_directories(target ...)` | เจาะจงว่า target ไหนมองเห็น include path นี้ |
| `link_libraries(...)` | `target_link_libraries(target ...)` | เจาะจงว่า target ไหน link กับ library ไหน |
| `set(CMAKE_CXX_FLAGS ...)` | `target_compile_options(target ...)` | เจาะจงว่า target ไหนได้รับ flag พิเศษ |
| `set(CMAKE_CXX_STANDARD 17)` (Global) | `target_compile_features(target PUBLIC cxx_std_17)` | เจาะจงมาตรฐานภาษาต่อ target |

> **กฎทองของ Part นี้**: เขียน CMake ใหม่ทุกครั้งให้ใช้คำสั่งตระกูล `target_*` เสมอ หลีกเลี่ยง
> คำสั่ง Global ตระกูลเก่า (`include_directories`, `link_libraries`, `add_definitions`)
> โดยสิ้นเชิง ยกเว้นกรณีพิเศษน้อยมากที่ต้องตั้งค่าระดับทั้งโปรเจกต์จริงๆ

### PUBLIC / PRIVATE / INTERFACE — คำสำคัญที่ควบคุมการ "แพร่กระจาย" ของค่า

คำสั่งตระกูล `target_*` ทุกตัวต้องระบุ **Visibility Keyword** หนึ่งใน 3 แบบ ซึ่งเป็นแนวคิดที่
สำคัญที่สุดของ Modern CMake:

| Keyword | ใครมองเห็น | ใช้เมื่อ |
|---|---|---|
| `PRIVATE` | ตัว target นี้เองเท่านั้น | ค่านั้นเป็นรายละเอียด**ภายใน**การ implement ของ target นี้ ไม่เกี่ยวกับใครที่มา link ต่อ |
| `PUBLIC` | ทั้งตัว target นี้ **และ** ทุก target ที่มา `target_link_libraries` กับมันในภายหลัง | ค่านั้นจำเป็นทั้งตอน compile target นี้เอง และตอนที่โค้ดคนอื่นเรียกใช้ target นี้ผ่าน header |
| `INTERFACE` | เฉพาะ target ที่มา link ต่อเท่านั้น (ตัวเองไม่ต้องใช้) | ใช้กับ Header-only Library ที่ตัว library เองไม่มีไฟล์ `.cpp` ให้ compile แต่ผู้ใช้ต้องได้ include path นี้ไปด้วย |

จะเห็นตัวอย่างการใช้งานจริงทั้ง 3 คำในหัวข้อถัดไป เมื่อสร้าง Library จริงๆ

---

## 91.5 สร้างและ Link Library ด้วย add_library (Step 725)

โปรเจกต์จริงแทบทุกโปรเจกต์แยกโค้ดเป็น Library (โมดูลที่ compile แยกและนำกลับมาใช้ซ้ำได้)
กับ Executable (โปรแกรมที่เรียกใช้ Library นั้น) — เหมือนกับ `math_utils` ใน Part 17 ที่แยก
`.h`/`.c` ออกจาก `main.c` แต่ตอนนี้เราจะให้ CMake จัดการเรื่อง compile/link ให้แทนการเขียน
คำสั่ง `gcc` ด้วยมือ

### โครงสร้างโปรเจกต์

```
mathutils_demo/
├── CMakeLists.txt
├── include/
│   └── mathutils/
│       └── mathutils.hpp
└── src/
    ├── mathutils.cpp
    └── main.cpp
```

`include/mathutils/mathutils.hpp`:

```cpp
#pragma once

namespace mathutils {

int add(int a, int b);
int gcd(int a, int b);

}  // namespace mathutils
```

`src/mathutils.cpp`:

```cpp
#include "mathutils/mathutils.hpp"

namespace mathutils {

int add(int a, int b) {
    return a + b;
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

`src/main.cpp`:

```cpp
#include <cstdio>
#include "mathutils/mathutils.hpp"

int main() {
    printf("add(2, 3)   = %d\n", mathutils::add(2, 3));
    printf("gcd(48, 18) = %d\n", mathutils::gcd(48, 18));
    return 0;
}
```

### CMakeLists.txt แบบ Target-based ที่สมบูรณ์

```cmake
cmake_minimum_required(VERSION 3.20)
project(MathUtilsDemo LANGUAGES CXX)

# --- สร้าง Static Library ชื่อ "mathutils" จาก mathutils.cpp ---
add_library(mathutils STATIC src/mathutils.cpp)

# ประกาศว่า include path นี้เป็น PUBLIC:
# - ตัว mathutils.cpp เองต้อง #include "mathutils/mathutils.hpp" ได้ (ใช้ตอน compile ตัวเอง)
# - executable ใดก็ตามที่มา target_link_libraries กับ mathutils จะได้ include path
#   นี้ติดมาด้วยอัตโนมัติ ไม่ต้องประกาศซ้ำ
target_include_directories(mathutils PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

# กำหนดว่า library นี้ต้องการ C++17 ขึ้นไป (PUBLIC = ผู้ที่ link ต่อก็ต้องใช้ C++17 ด้วย)
target_compile_features(mathutils PUBLIC cxx_std_17)

# --- สร้าง Executable ชื่อ "app" จาก main.cpp ---
add_executable(app src/main.cpp)

# link "app" เข้ากับ library "mathutils" แบบ PRIVATE
# (app ใช้ mathutils เป็นรายละเอียดภายในของตัวเอง ไม่มีใครมา link ต่อจาก app อีกที)
target_link_libraries(app PRIVATE mathutils)
```

### Configure และ Build จริง

```bash
mkdir build && cd build
cmake ..
```

ผลลัพธ์จริง:

```
-- The CXX compiler identification is GNU 13.3.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done (0.2s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/mathutils_demo/build
```

```bash
cmake --build .
```

```
[ 25%] Building CXX object CMakeFiles/mathutils.dir/src/mathutils.cpp.o
[ 50%] Linking CXX static library libmathutils.a
[ 50%] Built target mathutils
[ 75%] Building CXX object CMakeFiles/app.dir/src/main.cpp.o
[100%] Linking CXX executable app
[100%] Built target app
```

```bash
./app
```

```
add(2, 3)   = 5
gcd(48, 18) = 6
```

สังเกตสิ่งสำคัญ 2 จุด:

1. **`main.cpp` ไม่ได้ประกาศ `target_include_directories` ของตัวเองเลย** แต่ยัง
   `#include "mathutils/mathutils.hpp"` ได้ปกติ — เพราะ `PUBLIC` ของ `mathutils` "แพร่กระจาย"
   include path มาให้ `app` โดยอัตโนมัติผ่าน `target_link_libraries(app PRIVATE mathutils)`
   นี่คือพลังที่แท้จริงของ Modern CMake: **แค่ link กับ target ก็ได้ทุกอย่างที่ target นั้นประกาศ
   เป็น `PUBLIC` ติดมาด้วยเลย** ไม่ต้องมานั่งประกาศ include path ซ้ำที่ทุก executable ที่ใช้งาน
2. **`libmathutils.a`** ที่เกิดขึ้นคือ **Static Library** (เหมือนที่เรียนใน Part 39) — ถูกฝังรวม
   เข้าไปในไฟล์ `app` ตอน link เลย ไม่ต้องมีไฟล์ `.a` แถมไปด้วยตอน deploy

### ถ้าลืมใช้ PUBLIC แล้วใช้ PRIVATE แทนจะเกิดอะไรขึ้น

ลองเปลี่ยน `target_include_directories(mathutils PUBLIC ...)` เป็น `PRIVATE` แล้ว configure
ใหม่:

```cmake
target_include_directories(mathutils PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/include)
```

```bash
rm -rf build && mkdir build && cd build && cmake .. > /dev/null && cmake --build .
```

ผลลัพธ์จริง (compile `main.cpp` ล้มเหลวทันที):

```
[ 25%] Building CXX object CMakeFiles/mathutils.dir/src/mathutils.cpp.o
[ 50%] Linking CXX static library libmathutils.a
[ 50%] Built target mathutils
[ 75%] Building CXX object CMakeFiles/app.dir/src/main.cpp.o
/home/user/mathutils_demo/src/main.cpp:2:10: fatal error: mathutils/mathutils.hpp: No such file or directory
    2 | #include "mathutils/mathutils.hpp"
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
```

นี่คือหลักฐานที่ชัดเจนที่สุดว่า `PRIVATE` **ไม่แพร่กระจาย** ค่าไปยัง target ที่มา link ต่อ —
`mathutils.cpp` ยัง compile ผ่านได้ปกติ (เพราะตัวมันเองยังมองเห็น include path ของตัวเอง)
แต่ `app` ที่ link เข้ามาไม่ได้รับ include path นี้ติดมาด้วยอีกต่อไป ต้องเลือกใช้ `PUBLIC` ให้
ถูกต้องตามเจตนาการออกแบบเสมอ (กฎง่ายๆ: ถ้า header ของ library ถูก `#include` โดยไฟล์
ภายนอกที่มา link ต้องใช้ `PUBLIC` เท่านั้น)

---

## 91.6 จัดการหลายโฟลเดอร์ย่อยด้วย add_subdirectory (Step 726)

โปรเจกต์ตัวอย่างในหัวข้อ 91.5 ยังเก็บ `CMakeLists.txt` ไว้ไฟล์เดียวที่ root ซึ่งพอทนได้กับ
โปรเจกต์เล็ก แต่โปรเจกต์จริงระดับ production มักมีหลาย library ย่อยๆ ที่แต่ละตัวควรมี
`CMakeLists.txt` ของตัวเอง เพื่อให้จัดระเบียบและแยกความรับผิดชอบชัดเจน (เหมือนกับที่ Part 17
สอนเรื่องการแยก Module ด้วย Header/Linkage) `add_subdirectory` คือกลไกที่ทำให้ทำแบบนี้ได้

### โครงสร้างโปรเจกต์แบบหลายโฟลเดอร์

```
mathutils_multidir/
├── CMakeLists.txt              (root — ประกาศโปรเจกต์ แล้วเรียกโฟลเดอร์ย่อย)
├── libs/
│   └── mathutils/
│       ├── CMakeLists.txt      (นิยาม target "mathutils")
│       ├── include/mathutils/mathutils.hpp
│       └── src/mathutils.cpp
└── app/
    ├── CMakeLists.txt          (นิยาม target "app")
    └── main.cpp
```

`libs/mathutils/CMakeLists.txt` (สังเกตว่า**ไม่ต้อง**มี `cmake_minimum_required`/`project`
ซ้ำ — สองคำสั่งนี้มีแค่ที่ root เพียงจุดเดียวพอ):

```cmake
add_library(mathutils STATIC src/mathutils.cpp)
target_include_directories(mathutils PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)
target_compile_features(mathutils PUBLIC cxx_std_17)
```

`app/CMakeLists.txt`:

```cmake
add_executable(app main.cpp)
target_link_libraries(app PRIVATE mathutils)
```

`CMakeLists.txt` ที่ root — ไฟล์นี้คือจุดเดียวที่ต้องมี `cmake_minimum_required` และ `project`
แล้วใช้ `add_subdirectory` ดึงโฟลเดอร์ย่อยเข้ามาประมวลผลตามลำดับ:

```cmake
cmake_minimum_required(VERSION 3.20)
project(MathUtilsMultiDir LANGUAGES CXX)

add_subdirectory(libs/mathutils)   # ประมวลผล libs/mathutils/CMakeLists.txt ก่อน
add_subdirectory(app)              # แล้วค่อยประมวลผล app/CMakeLists.txt
```

สังเกตว่า `add_subdirectory(app)` ต้องมาหลัง `add_subdirectory(libs/mathutils)` เพราะ
`app/CMakeLists.txt` เรียก `target_link_libraries(app PRIVATE mathutils)` ซึ่ง target ชื่อ
`mathutils` ต้องถูกประกาศไว้ก่อนแล้วเท่านั้น (CMake อ่านและประมวลผลไฟล์ตามลำดับที่เจอ
`add_subdirectory` เหมือนอ่านโค้ดจากบนลงล่าง)

### Configure และ Build จริง

```bash
mkdir build && cd build
cmake ..
```

ผลลัพธ์จริง:

```
-- The CXX compiler identification is GNU 13.3.0
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done (0.2s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/mathutils_multidir/build
```

```bash
cmake --build .
```

```
[ 25%] Building CXX object libs/mathutils/CMakeFiles/mathutils.dir/src/mathutils.cpp.o
[ 50%] Linking CXX static library libmathutils.a
[ 50%] Built target mathutils
[ 75%] Building CXX object app/CMakeFiles/app.dir/main.cpp.o
[100%] Linking CXX executable app
[100%] Built target app
```

```bash
./app/app
```

```
add(2, 3)   = 5
gcd(48, 18) = 6
```

สังเกตว่าโครงสร้างโฟลเดอร์ภายใน `build/` (`libs/mathutils/CMakeFiles/...`, `app/CMakeFiles/...`)
**สะท้อนโครงสร้างโฟลเดอร์ source ทุกประการ** — CMake สร้างโฟลเดอร์ย่อยใน `build/` ให้ตรงกับ
ตำแหน่งของแต่ละ `add_subdirectory` โดยอัตโนมัติ ทำให้แม้โปรเจกต์จะมี library ย่อยเป็นสิบๆ ตัว
โครงสร้างก็ยังอ่านและดูแลรักษาได้ง่าย ทีมใหญ่ที่แบ่งงานกันดูแลคนละ library สามารถแก้ไข
`CMakeLists.txt` ของโฟลเดอร์ตัวเองได้โดยไม่ชนกับใคร

> **แนวปฏิบัติมาตรฐานในโปรเจกต์จริง**: โปรเจกต์ C++ ระดับ production แทบทุกโปรเจกต์ (เช่น
> LLVM, OpenCV) จะมีโครงสร้างแบบนี้เป๊ะๆ — มีโฟลเดอร์ `CMakeLists.txt` ที่ root เพียงไฟล์เดียว
> ทำหน้าที่ "orchestrate" เรียก `add_subdirectory` ของแต่ละ module/library ย่อยเข้ามารวมกัน

---

## 91.7 CMAKE_BUILD_TYPE: Debug กับ Release (Step 727)

ใน Part 89 เราเห็นแล้วว่า optimization level ของ compiler (`-O0` เทียบกับ `-O2`) ส่งผลต่อ
ประสิทธิภาพของโปรแกรมอย่างมหาศาล และใน Part 90 ก็เน้นย้ำว่าต้อง benchmark ที่ `-O2` ขึ้นไป
เสมอ CMake มีกลไกมาตรฐานสำหรับสลับระหว่างโหมด **Debug** (ไว้ debug ระหว่างพัฒนา) กับ
**Release** (ไว้ deploy จริง) ผ่านตัวแปร `CMAKE_BUILD_TYPE`

### ค่า CMAKE_BUILD_TYPE มาตรฐานที่ CMake รู้จักในตัว

| ค่า | Flag ที่ CMake ใส่ให้อัตโนมัติ (GCC/Clang) | ใช้เมื่อ |
|---|---|---|
| `Debug` | `-g` (ใส่ debug symbol, ไม่ optimize) | กำลังพัฒนา/debug ด้วย GDB (Part 37) |
| `Release` | `-O3 -DNDEBUG` (optimize เต็มที่, ปิด `assert`) | Build เพื่อ deploy จริง |
| `RelWithDebInfo` | `-O2 -g -DNDEBUG` | Release แต่ยังอยากมี debug symbol ไว้วิเคราะห์ปัญหาที่เกิดใน production |
| `MinSizeRel` | `-Os -DNDEBUG` | เน้นให้ไฟล์ executable มีขนาดเล็กที่สุด (สำคัญมากกับงาน Embedded — Part 117) |

### พิสูจน์ด้วยการดู compile command จริง

ใช้โปรเจกต์ `HelloCMake` จากหัวข้อ 91.1 กำหนด `CMAKE_BUILD_TYPE` ตอน configure ผ่าน
`-D` แล้วเปิดโหมด verbose เพื่อดูคำสั่ง compile ที่แท้จริงที่ CMake สร้างให้:

```bash
mkdir build_debug && cd build_debug
cmake -DCMAKE_BUILD_TYPE=Debug ..
make VERBOSE=1 | grep "c++.*main.cpp"
```

ผลลัพธ์จริง:

```
/usr/bin/c++   -g -MD -MT CMakeFiles/hello.dir/main.cpp.o -MF CMakeFiles/hello.dir/main.cpp.o.d -o CMakeFiles/hello.dir/main.cpp.o -c main.cpp
```

```bash
cd .. && mkdir build_release && cd build_release
cmake -DCMAKE_BUILD_TYPE=Release ..
make VERBOSE=1 | grep "c++.*main.cpp"
```

ผลลัพธ์จริง:

```
/usr/bin/c++   -O3 -DNDEBUG -MD -MT CMakeFiles/hello.dir/main.cpp.o -MF CMakeFiles/hello.dir/main.cpp.o.d -o CMakeFiles/hello.dir/main.cpp.o -c main.cpp
```

เห็นความต่างชัดเจนจากคำสั่งจริงที่ CMake ส่งให้ compiler: `-g` เทียบกับ `-O3 -DNDEBUG` — เรา
**ไม่ต้องจำ flag พวกนี้เอง** เหมือนตอนเขียน Makefile ใน Part 18 ที่ต้องเขียน `ifeq/else/endif`
สลับ flag ด้วยมือ (หัวข้อ 18 แบบฝึกหัดข้อ 2) CMake จัดการให้อัตโนมัติผ่านตัวแปรมาตรฐานตัวเดียว

### ข้อควรระวัง: build folder แยกกันตาม Build Type เสมอ

สังเกตว่าตัวอย่างข้างบนใช้ `build_debug/` กับ `build_release/` **แยกโฟลเดอร์กัน** — นี่คือ
แนวทางที่ถูกต้องเพราะ CMake (เมื่อใช้ Generator แบบ Single-config อย่าง Make/Ninja) จะจำ
`CMAKE_BUILD_TYPE` ไว้ใน `CMakeCache.txt` ของ build folder นั้นๆ **ตอน configure ครั้งแรก
เท่านั้น** ถ้าใช้ build folder เดียวกันแล้วพยายามสลับ `-DCMAKE_BUILD_TYPE` ไปมาโดยไม่ลบ
`CMakeCache.txt` ก่อน อาจได้ผลลัพธ์ที่งงว่าทำไม flag ไม่เปลี่ยนตามที่คาดหวัง — วิธีที่ปลอดภัย
ที่สุดคือแยกโฟลเดอร์ build ตาม configuration แบบนี้เสมอ หรือลบ `build/` ทิ้งแล้ว configure ใหม่
ทุกครั้งที่ต้องการเปลี่ยน Build Type

> **หมายเหตุ**: Generator แบบ Multi-config (เช่น Visual Studio, Xcode) ไม่มีปัญหานี้ เพราะ
> เก็บทั้ง Debug และ Release ไว้ในโปรเจกต์เดียวกันได้ แล้วเลือกตอน build ด้วย
> `cmake --build . --config Release` แทน — เป็นอีกเหตุผลหนึ่งที่ CMake ได้เปรียบ Makefile
> ตรงๆ มาก เพราะรองรับแนวคิดทั้งสองแบบในระบบเดียว

---

## 91.8 Port โปรเจกต์ math_utils จาก Makefile สู่ CMake (Step 728)

ถึงเวลานำทุกเทคนิคที่เรียนมาประกอบร่างเป็นงานจริง: port โปรเจกต์ `math_utils` จาก **Part 18**
(ที่เดิม build ด้วย Makefile) มาใช้ CMake แทนทั้งหมด โดยใช้ไฟล์ source ชุดเดียวกันทุกตัวอักษร
ไม่แก้โค้ด C แม้แต่บรรทัดเดียว — เปลี่ยนแค่เครื่องมือ build เท่านั้น

### ไฟล์ source (เหมือน Part 18 ทุกประการ)

`math_utils.h`:

```c
#ifndef MATH_UTILS_H
#define MATH_UTILS_H

#include <stddef.h> /* size_t */

int  mu_add(int a, int b);
int  mu_subtract(int a, int b);
long mu_power(int base, unsigned int exponent);
int  mu_gcd(int a, int b);
int  mu_is_prime(int n);
double mu_average(const int *values, size_t count);

extern long mu_call_count;

void mu_reset_call_count(void);
long mu_get_call_count(void);

#endif /* MATH_UTILS_H */
```

`math_utils.c` (ตัด comment ส่วนหัวออกเพื่อประหยัดพื้นที่ เนื้อหาฟังก์ชันเหมือน Part 17-18
ทุกประการ):

```c
#include "math_utils.h"

long mu_call_count = 0;

static void mu_track_call(void) {
    mu_call_count++;
}

static int mu_abs_int(int x) {
    return (x < 0) ? -x : x;
}

int mu_add(int a, int b) {
    mu_track_call();
    return a + b;
}

int mu_subtract(int a, int b) {
    mu_track_call();
    return a - b;
}

long mu_power(int base, unsigned int exponent) {
    mu_track_call();
    long result = 1;
    for (unsigned int i = 0; i < exponent; i++) {
        result *= base;
    }
    return result;
}

int mu_gcd(int a, int b) {
    mu_track_call();
    a = mu_abs_int(a);
    b = mu_abs_int(b);
    while (b != 0) {
        int temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}

int mu_is_prime(int n) {
    mu_track_call();
    if (n < 2) {
        return 0;
    }
    for (int i = 2; (long)i * i <= n; i++) {
        if (n % i == 0) {
            return 0;
        }
    }
    return 1;
}

double mu_average(const int *values, size_t count) {
    mu_track_call();
    if (count == 0) {
        return 0.0;
    }
    long sum = 0;
    for (size_t i = 0; i < count; i++) {
        sum += values[i];
    }
    return (double)sum / (double)count;
}

void mu_reset_call_count(void) {
    mu_call_count = 0;
}

long mu_get_call_count(void) {
    return mu_call_count;
}
```

`main.c`:

```c
#include <stdio.h>
#include "math_utils.h"

int main(void) {
    printf("mu_add(2, 3)      = %d\n", mu_add(2, 3));
    printf("mu_subtract(10,4) = %d\n", mu_subtract(10, 4));
    printf("mu_power(2, 10)   = %ld\n", mu_power(2, 10));
    printf("mu_gcd(48, 18)    = %d\n", mu_gcd(48, 18));
    printf("mu_is_prime(17)   = %d\n", mu_is_prime(17));

    int scores[] = {80, 90, 70, 100, 60};
    size_t n = sizeof(scores) / sizeof(scores[0]);
    printf("mu_average(...)   = %.2f\n", mu_average(scores, n));

    printf("เรียกฟังก์ชันใน module ไปทั้งหมด %ld ครั้ง\n", mu_get_call_count());

    return 0;
}
```

### CMakeLists.txt แทนที่ Makefile ทั้งไฟล์

เทียบกับ Makefile ฉบับสมบูรณ์ใน Part 18 หัวข้อ 18.8 (ที่ใช้ `wildcard`, `patsubst`,
`-MMD -MP` รวม 30 บรรทัด) นี่คือ CMake เวอร์ชันที่ทำงานเดียวกันทุกประการ:

```cmake
cmake_minimum_required(VERSION 3.20)
project(MathUtilsPort C)

set(CMAKE_C_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED ON)

# --- Library: math_utils ---
add_library(math_utils STATIC math_utils.c)
target_include_directories(math_utils PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_options(math_utils PRIVATE -Wall -Wextra -Wpedantic)

# --- Executable: program ---
add_executable(program main.c)
target_link_libraries(program PRIVATE math_utils)
target_compile_options(program PRIVATE -Wall -Wextra -Wpedantic)
```

สังเกตว่า `project(MathUtilsPort C)` ระบุภาษาเป็น `C` (ไม่ใช่ `CXX`) เพราะโปรเจกต์นี้เป็น C
ล้วนตามต้นฉบับ — CMake รองรับทั้ง C และ C++ (และภาษาอื่นๆ อีกหลายภาษา) ในระบบเดียวกัน

### Configure, Build, และ Run จริง

```bash
mkdir build && cd build
cmake ..
```

```
-- The C compiler identification is GNU 13.3.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Configuring done (0.2s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/math_utils_cmake/build
```

```bash
cmake --build .
```

```
[ 25%] Building C object CMakeFiles/math_utils.dir/math_utils.c.o
[ 50%] Linking C static library libmath_utils.a
[ 50%] Built target math_utils
[ 75%] Building C object CMakeFiles/program.dir/main.c.o
[100%] Linking C executable program
[100%] Built target program
```

```bash
./program
```

```
mu_add(2, 3)      = 5
mu_subtract(10,4) = 6
mu_power(2, 10)   = 1024
mu_gcd(48, 18)    = 6
mu_is_prime(17)   = 1
mu_average(...)   = 80.00
เรียกฟังก์ชันใน module ไปทั้งหมด 6 ครั้ง
```

ผลลัพธ์**ตรงกันทุกตัวอักษร**กับที่ได้จาก Makefile ใน Part 18 — พิสูจน์ว่า CMake สร้างโปรแกรม
ที่ทำงานเหมือนกันทุกประการ เพียงแค่เปลี่ยนเครื่องมือ build เท่านั้น

### พิสูจน์ว่า CMake ติดตาม Header Dependency ให้อัตโนมัติ — ไม่ต้องใช้ -MMD -MP เอง

จำได้ไหมว่าใน Part 18 หัวข้อ 18.5 เราเจอปัญหาว่า **Pattern Rule เพียวๆ ไม่รู้จัก dependency
ของ header** ต้องแก้ด้วยเทคนิค `-MMD -MP` ในหัวข้อ 18.7 เอง มาทดสอบว่า CMake จัดการเรื่องนี้
ให้เราแล้วหรือยัง โดยสั่ง build อีกครั้งแบบไม่แก้ไขอะไร (ควรข้ามทุกอย่าง) แล้วลอง `touch`
header ดู:

```bash
$ cmake --build .
```

```
[ 50%] Built target math_utils
[100%] Built target program
```

CMake ตรวจสอบแล้วว่าไม่มีอะไรเปลี่ยน จึงข้ามการ build ทั้งหมด (เหมือนพฤติกรรมพื้นฐานของ
Make) ทีนี้ลอง `touch` header ดูว่า CMake ตรวจจับการเปลี่ยนแปลงของ header ได้เองหรือไม่
**โดยที่เราไม่ได้เขียนอะไรเกี่ยวกับ dependency tracking ไว้ใน `CMakeLists.txt` เลย**:

```bash
$ touch ../math_utils.h
$ cmake --build .
```

ผลลัพธ์จริง:

```
[ 25%] Building C object CMakeFiles/math_utils.dir/math_utils.c.o
[ 50%] Linking C static library libmath_utils.a
[ 50%] Built target math_utils
[ 75%] Building C object CMakeFiles/program.dir/main.c.o
[100%] Linking C executable program
[100%] Built target program
```

ทั้ง `math_utils.c` และ `main.c` (ทั้งสองไฟล์ `#include "math_utils.h"`) ถูกคอมไพล์ใหม่ทันที
โดยที่เราไม่ต้องเขียน flag `-MMD -MP` หรือจัดการไฟล์ `.d` เองเลยแม้แต่นิดเดียว — CMake
สร้าง build system ที่ทำ **Automatic Dependency Scanning** ให้เป็นค่า default อยู่แล้ว
(เบื้องหลังคือ Makefile ที่ CMake generate มีกลไกคล้ายกับ `-MMD -MP` ฝังอยู่ในตัวโดยอัตโนมัติ
สำหรับทุก target) นี่คือหนึ่งในเหตุผลสำคัญที่สุดที่ทำให้ CMake ลดภาระของนักพัฒนาไปได้มาก
เมื่อเทียบกับการเขียน Makefile ด้วยมือ

### ตารางเปรียบเทียบสรุป: สิ่งที่ทำเองใน Makefile vs สิ่งที่ CMake ทำให้อัตโนมัติ

| งาน | Makefile (Part 18) | CMake |
|---|---|---|
| ระบุไฟล์ source ทั้งหมด | ต้องใช้ `$(wildcard *.c)` เอง | ระบุตรงๆ ใน `add_executable`/`add_library` (หรือใช้ `file(GLOB ...)` แม้ผู้เชี่ยวชาญ CMake จำนวนมากแนะนำให้เขียนชื่อไฟล์ตรงๆ เพื่อความชัดเจน) |
| แปลง `.c` เป็น `.o` | ต้องเขียน Pattern Rule `%.o: %.c` เอง | ทำให้อัตโนมัติทั้งหมด ไม่ต้องเขียน rule เอง |
| ติดตาม dependency ของ header | ต้องใช้ `-MMD -MP` + `-include $(DEPS)` เอง | ทำอัตโนมัติ ไม่ต้องตั้งค่าอะไรเพิ่ม |
| สลับ Debug/Release | ต้องเขียน `ifeq/else/endif` เอง | ตั้งค่า `CMAKE_BUILD_TYPE` ตัวเดียวจบ |
| Build ข้าม Platform (Windows/macOS) | ต้องเขียน Makefile แยกหรือปรับแก้เอง | CMake จัดการให้ผ่าน Generator ที่เหมาะกับแต่ละ OS |
| Build ด้วย IDE (Visual Studio/Xcode) | ทำไม่ได้เลยโดยตรง | `cmake -G "Visual Studio 17 2022"` / `-G Xcode` |

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ทำ In-Source Build** — สั่ง `cmake .` ตรงๆ ในโฟลเดอร์ source โดยไม่สร้าง `build/` แยก
   ทำให้ไฟล์ที่ generate ปนกับ source code (ดังที่พิสูจน์ในหัวข้อ 91.3) เสี่ยง commit ไฟล์ขยะ
   เข้า Git และทำ clean build ได้ยาก ควรใช้ `mkdir build && cd build && cmake ..` เป็น
   ธรรมเนียมตายตัวเสมอ
2. **ใช้คำสั่ง Global แบบเก่า (`include_directories`, `link_libraries`) แทนคำสั่ง `target_*`** —
   ทำให้การตั้งค่าไม่ชัดเจนว่ามีผลกับ target ไหนบ้าง เมื่อโปรเจกต์โตขึ้นจะไล่ debug ยากมาก ควร
   ใช้ `target_include_directories`, `target_link_libraries`, `target_compile_features` เสมอ
3. **ลืมระบุ `PUBLIC`/`PRIVATE`/`INTERFACE` ให้ถูกต้อง** — โดยเฉพาะการใช้ `PRIVATE` ทั้งที่
   ควรเป็น `PUBLIC` ทำให้ target ที่มา link ต่อมองไม่เห็น include path หรือ compile feature
   ที่จำเป็น เกิด error แบบ `fatal error: ... No such file or directory` ทั้งที่ library build
   ผ่านปกติ (ดูตัวอย่างจริงในหัวข้อ 91.5)
4. **ใช้ `CMakeCache.txt` ร่วมกันข้ามเครื่อง/ข้าม configuration** — `CMakeCache.txt` เก็บ
   absolute path และค่าที่ configure ไว้เฉพาะเครื่องนั้นๆ ห้าม commit เข้า Git เด็ดขาด (ต้อง
   ใส่ `build/` ทั้งโฟลเดอร์ไว้ใน `.gitignore`) และห้ามสลับ `CMAKE_BUILD_TYPE` ใน build folder
   เดียวกันโดยไม่ลบ cache ก่อน (ดูหัวข้อ 91.7)
5. **ลืม `cmake_minimum_required` หรือวางไว้ไม่ใช่บรรทัดแรก** — ทำให้ CMake ใช้ Policy
   เวอร์ชันที่ไม่แน่นอน อาจได้พฤติกรรมของคำสั่งบางตัวที่ต่างไปจากที่คาดหวัง (โดยเฉพาะเมื่อ
   นำ `CMakeLists.txt` เก่าไปรันบน CMake เวอร์ชันใหม่กว่าที่เขียนไว้ตอนแรกมาก)
6. **ลำดับ `add_subdirectory` ผิด** — เรียก `add_subdirectory` ของโฟลเดอร์ที่ใช้ `target_link_libraries`
   อ้างถึง target ก่อนที่ target นั้นจะถูกประกาศ ทำให้ได้ error ประเภท "target ... not found"
   ต้องเรียงลำดับ `add_subdirectory` ให้ library ถูกประกาศก่อนโฟลเดอร์ที่จะมา link ใช้งานเสมอ

---

## แบบฝึกหัดท้ายบท

1. สร้างโปรเจกต์ CMake ใหม่ที่มี Static Library ชื่อ `string_utils` (มีฟังก์ชัน `reverse` และ
   `is_palindrome` เหมือนโปรเจกต์ `string_utils` จาก Part 17 แบบฝึกหัดข้อ 1) และ executable
   ที่เรียกใช้ library นี้ ต้องใช้คำสั่งตระกูล `target_*` เท่านั้น ห้ามใช้คำสั่ง Global แบบเก่า
2. ปรับโปรเจกต์จากข้อ 1 ให้เป็นแบบหลายโฟลเดอร์ย่อยด้วย `add_subdirectory` (แยก library กับ
   executable คนละโฟลเดอร์เหมือนหัวข้อ 91.6)
3. ทดลองตั้งค่า `CMAKE_BUILD_TYPE=RelWithDebInfo` กับโปรเจกต์ใดก็ได้ที่เขียนไว้ แล้วใช้
   `make VERBOSE=1` ดูว่า flag ที่ compiler ได้รับจริงๆ คืออะไร (เทียบกับที่ตารางในหัวข้อ 91.7
   บอกไว้ว่าควรเป็น `-O2 -g -DNDEBUG`)
4. ทดลองลบ `target_compile_features(mathutils PUBLIC cxx_std_17)` ออกจากโปรเจกต์ในหัวข้อ
   91.5 แล้วลองเขียนโค้ดที่ใช้ฟีเจอร์ C++17 (เช่น Structured Bindings จาก Part 72) ใน
   `main.cpp` สังเกตว่าเกิด error หรือไม่ อธิบายว่าทำไม
5. เขียน `CMakeLists.txt` ที่มี 2 executable ในโปรเจกต์เดียว (เช่น `app_debug_demo` และ
   `app_release_demo`) ที่ต่างกันแค่ `target_compile_options` (ตัวหนึ่งได้ `-O0 -g` อีกตัวได้
   `-O3`) โดยไม่ใช้ `CMAKE_BUILD_TYPE` เลย (ใบ้: ใช้ `target_compile_options` แยกต่อ target
   ได้โดยตรง)
6. (โบนัส) ค้นคว้าเพิ่มเติมว่าคำสั่ง `install(TARGETS ...)` ของ CMake ใช้ทำอะไร แล้วลองเพิ่ม
   เข้าไปในโปรเจกต์จากข้อ 1 เพื่อให้สั่ง `cmake --install build --prefix ~/myapp` แล้วได้ไฟล์
   executable ไปอยู่ที่ `~/myapp/bin/` (คล้ายกับ target `install` ที่เขียนเองใน Part 18
   แบบฝึกหัดข้อ 5 แต่เป็นกลไกมาตรฐานของ CMake เอง)

### แนวทางเฉลยข้อ 1

`string_utils.hpp`:

```cpp
#pragma once
#include <string>

namespace strutils {

std::string reverse(const std::string& s);
bool is_palindrome(const std::string& s);

}  // namespace strutils
```

`string_utils.cpp`:

```cpp
#include "string_utils.hpp"
#include <algorithm>
#include <cctype>

namespace strutils {

std::string reverse(const std::string& s) {
    std::string result = s;
    std::reverse(result.begin(), result.end());
    return result;
}

bool is_palindrome(const std::string& s) {
    std::string normalized;
    for (char c : s) {
        if (std::isalpha(static_cast<unsigned char>(c))) {
            normalized += static_cast<char>(std::tolower(static_cast<unsigned char>(c)));
        }
    }
    std::string rev = normalized;
    std::reverse(rev.begin(), rev.end());
    return normalized == rev;
}

}  // namespace strutils
```

`main.cpp`:

```cpp
#include <cstdio>
#include "string_utils.hpp"

int main() {
    printf("reverse(\"Hello\") = \"%s\"\n", strutils::reverse("Hello").c_str());
    printf("is_palindrome(\"level\")   = %d\n", strutils::is_palindrome("level"));
    printf("is_palindrome(\"Racecar\") = %d\n", strutils::is_palindrome("Racecar"));
    printf("is_palindrome(\"Hello\")   = %d\n", strutils::is_palindrome("Hello"));
    return 0;
}
```

`CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.20)
project(StringUtilsDemo LANGUAGES CXX)

add_library(string_utils STATIC string_utils.cpp)
target_include_directories(string_utils PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_compile_features(string_utils PUBLIC cxx_std_17)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE string_utils)
```

ทดสอบ:

```bash
$ mkdir build && cd build && cmake .. > /dev/null && cmake --build .
[ 25%] Building CXX object CMakeFiles/string_utils.dir/string_utils.cpp.o
[ 50%] Linking CXX static library libstring_utils.a
[ 50%] Built target string_utils
[ 75%] Building CXX object CMakeFiles/app.dir/main.cpp.o
[100%] Linking CXX executable app
[100%] Built target app

$ ./app
reverse("Hello") = "olleH"
is_palindrome("level")   = 1
is_palindrome("Racecar") = 1
is_palindrome("Hello")   = 0
```

โน้ตสำคัญ: ไฟล์นี้ใช้คำสั่งตระกูล `target_*` ล้วน ไม่มีคำสั่ง Global เลยแม้แต่บรรทัดเดียว
ตรงตามข้อกำหนดของโจทย์

### แนวทางเฉลยข้อ 5

```cmake
cmake_minimum_required(VERSION 3.20)
project(TwoConfigDemo LANGUAGES CXX)

add_executable(app_debug_demo main.cpp)
target_compile_options(app_debug_demo PRIVATE -O0 -g)

add_executable(app_release_demo main.cpp)
target_compile_options(app_release_demo PRIVATE -O3)
```

ทดสอบด้วย `VERBOSE=1` เพื่อยืนยันว่า flag ต่างกันจริงในแต่ละ target ทั้งที่มาจาก
`CMakeLists.txt` ไฟล์เดียวกันและไม่ได้ใช้ `CMAKE_BUILD_TYPE` เลย:

```bash
$ mkdir build && cd build && cmake .. > /dev/null && make VERBOSE=1 2>&1 | grep "c++.*main.cpp"
/usr/bin/c++    -O0 -g -MD -MT CMakeFiles/app_debug_demo.dir/main.cpp.o ... -c main.cpp
/usr/bin/c++    -O3 -MD -MT CMakeFiles/app_release_demo.dir/main.cpp.o ... -c main.cpp
```

ข้อสังเกตสำคัญจากแบบฝึกหัดนี้: `target_compile_options` ให้ความยืดหยุ่นระดับ**ต่อ target**
มากกว่า `CMAKE_BUILD_TYPE` ซึ่งเป็นการตั้งค่าระดับ**ทั้งโปรเจกต์** — ในทางปฏิบัติ โปรเจกต์
ส่วนใหญ่นิยมใช้ `CMAKE_BUILD_TYPE` เป็นหลักเพราะเรียบง่ายกว่าและเป็นธรรมเนียมสากลที่ IDE/
CI ทุกตัวรู้จัก แต่การเข้าใจว่า `target_compile_options` ทำอะไรได้ละเอียดกว่าเป็นความรู้สำคัญ
เมื่อต้องเจอโปรเจกต์ที่มี target หลายตัวที่ต้องการ optimization level ต่างกันในการ build ครั้ง
เดียวกันจริงๆ (เช่น executable หลักต้องการ optimize เต็มที่ แต่ executable สำหรับ debug
เครื่องมือภายในทีมไม่จำเป็นต้อง optimize)

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่า CMake ไม่ใช่ Build System แต่เป็น **Meta-Build-System** ที่ generate ไฟล์ build
  จริง (Makefile, Ninja, Visual Studio Solution) จาก `CMakeLists.txt` ไฟล์เดียว และพิสูจน์
  ด้วยการรันจริงว่า `CMakeLists.txt` เดียวกันสามารถ generate ได้ทั้ง Makefile และ Ninja
  รวมถึงสลับ compiler จาก GCC เป็น Clang ได้โดยไม่แก้โค้ดเลย
- เขียน `CMakeLists.txt` พื้นฐานด้วย `cmake_minimum_required`, `project`, `add_executable`
  และฝึกวินัย **Out-of-Source Build** พร้อมเห็นผลจริงว่า In-Source Build ทำให้ source tree
  ปนเปื้อนอย่างไร
- เข้าใจแนวคิด **Target-based** ของ Modern CMake และความหมายของ `PUBLIC`/`PRIVATE`/
  `INTERFACE` ที่ควบคุมการแพร่กระจายของ include path/library/compile feature ระหว่าง target
- สร้าง Static Library ด้วย `add_library` และ link เข้ากับ Executable ด้วย
  `target_link_libraries` พร้อมพิสูจน์ผลของการเลือกใช้ `PUBLIC` ผิดเป็น `PRIVATE`
- จัดการโปรเจกต์หลายโฟลเดอร์ด้วย `add_subdirectory` ให้โครงสร้างสอดคล้องกับ source tree
- ใช้ `CMAKE_BUILD_TYPE` สลับ Debug/Release และเห็นด้วยตาตัวเองว่า flag ของ compiler
  (`-g` เทียบกับ `-O3 -DNDEBUG`) เปลี่ยนไปจริงตามที่ตั้งค่า
- Port โปรเจกต์ `math_utils` จาก Part 18 มาใช้ CMake ได้สำเร็จ 100% (ผลลัพธ์ตรงกันทุก
  ตัวอักษร) พร้อมพิสูจน์ว่า CMake ติดตาม dependency ของ header ให้อัตโนมัติโดยไม่ต้องเขียน
  `-MMD -MP` เองเหมือน Makefile

CMake คือรากฐานสำคัญที่สุดของ Module H เพราะทุกเครื่องมือที่จะเรียนต่อจากนี้ ไม่ว่าจะเป็น
Package Manager (Part 92), Unit Testing Framework (Part 93), CI/CD (Part 94), Static
Analysis (Part 95), หรือ Sanitizer (Part 96) ล้วน**เชื่อมต่อเข้ากับโปรเจกต์ผ่าน CMake**
เป็นหลักทั้งสิ้น ใน **Part 92** เราจะเรียนรู้วิธีจัดการ 3rd-party library ในโปรเจกต์ C++ ด้วย
**Package Manager** สมัยใหม่อย่าง **Conan** และ **vcpkg** ซึ่งใช้ CMake เป็นกลไกเชื่อมต่อหลัก
เช่นกัน

**ต่อไป:** [Part 92 — Package Management: Conan และ vcpkg](./part-092-package-management.md)
