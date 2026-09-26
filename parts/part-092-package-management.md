# Part 92: Package Management: Conan และ vcpkg (Step 729–736)

> Module H — Build Systems, Testing และ Tooling | Part 92 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 729–736
> Part ก่อนหน้า: [Part 91 — CMake ตั้งแต่พื้นฐานถึงขั้นสูง](./part-091-cmake.md) | Part ถัดไป: [Part 93 — Unit Testing ด้วย Google Test และ Catch2](./part-093-unit-testing.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายได้ว่าทำไมการหา/build/link 3rd-party library ด้วยมือใน C++ ถึงเป็นปัญหาที่ยากกว่า
   ภาษาอื่นที่มี Package Manager มาตรฐานอย่าง npm (JavaScript) หรือ pip (Python) มาก
2. อธิบายแนวคิดของ **vcpkg** (พัฒนาโดย Microsoft) และ **Conan** (cross-platform, เน้น
   dependency resolution) พร้อมเปรียบเทียบจุดเด่น-จุดด้อยของทั้งสองเครื่องมือ
3. ติดตั้งและทดลองใช้งาน vcpkg/Conan จริง พร้อมเข้าใจ syntax และ workflow มาตรฐานที่ใช้
   เชื่อมต่อกับ CMake ผ่าน `find_package`
4. อธิบายได้ว่าทำไมความพยายาม install package ผ่าน network ใน sandbox ของบทเรียนนี้
   ล้มเหลว และเข้าใจว่าความล้มเหลวแบบนี้สะท้อนปัญหาจริงอะไรบ้างในโลกของ CI/CD
5. เปรียบเทียบแนวทาง Package Manager สมัยใหม่ (vcpkg/Conan) กับแนวทาง System Package
   (`apt install libxxx-dev`) ที่ใช้มาตลอดหลักสูตรนี้ ในแง่ Portability และ Version Pinning
6. เลือกแนวทางจัดการ Dependency ที่เหมาะสมกับสถานการณ์งานจริงแต่ละแบบได้อย่างมีเหตุผล

---

## 92.1 ปัญหาที่ Package Manager แก้ (Step 729)

ตลอดหลักสูตรนี้ เวลาต้องการใช้ library ภายนอก เราใช้วิธีเดียวมาตลอด: `sudo apt install
libxxx-dev` เช่น `libbenchmark-dev` ใน Part 90 หรือ `libgtest-dev` ที่จะใช้ใน Part 93 วิธีนี้
ใช้งานได้ดีเพราะ Ubuntu/Debian มี Package สำเร็จรูปให้เกือบทุกไลบรารีที่มีชื่อเสียง แต่พอโปรเจกต์
ต้อง build ข้าม Platform หรือใช้เวอร์ชันเฉพาะเจาะจงของ library ปัญหาที่ซ่อนอยู่จะเริ่มปรากฏ:

1. **Ubuntu กับ macOS กับ Windows ไม่มี Package Manager ร่วมกัน**: `apt` มีเฉพาะบน
   Debian/Ubuntu, macOS ใช้ `brew`, Windows ไม่มีอะไรมาตรฐานให้ในตัวเลย ชื่อ package
   บางตัวก็เขียนไม่เหมือนกันข้าม distro (`libssl-dev` บน Ubuntu อาจชื่อ `openssl-devel` บน
   Fedora) ทำให้สคริปต์ setup โปรเจกต์ต้องเขียนแยกเป็นหลายเวอร์ชันตาม OS
2. **เวอร์ชันของ library ถูกกำหนดโดย OS ไม่ใช่โดยโปรเจกต์**: `apt install libfmt-dev` บน
   Ubuntu 24.04 จะได้เวอร์ชันที่ทีม Ubuntu เลือกให้ตายตัว ถ้าโปรเจกต์ต้องการ library เวอร์ชัน
   เฉพาะ (เช่น เพื่อหลีกเลี่ยงบั๊กที่แก้ในเวอร์ชันใหม่กว่า หรือต้องการ feature ที่เพิ่งเพิ่มเข้ามา)
   จะทำไม่ได้เลยถ้าพึ่ง system package อย่างเดียว ต้อง build จาก source เองด้วยมือ
3. **ไม่มีการล็อกเวอร์ชันแบบ Reproducible Build**: ในภาษาอย่าง Node.js มีไฟล์
   `package-lock.json` ที่ล็อกทุก dependency ให้ตรงกันทุกเครื่องเป๊ะๆ แต่การเขียน
   `find_package` ตรงๆ ใน CMake โดยพึ่ง system package ไม่มีกลไกล็อกเวอร์ชันแบบนี้ในตัว
   ทีมงาน 2 คนอาจได้ library คนละเวอร์ชันโดยไม่รู้ตัว ทำให้เกิดบั๊กที่เกิดเฉพาะบางเครื่อง
4. **การ build library ที่ไม่มี Package สำเร็จรูปเป็นเรื่องเจ็บปวด**: library บางตัว (โดยเฉพาะ
   library เฉพาะทางหรือใหม่มาก) ไม่มีใน apt repository เลย ต้อง clone source, อ่าน README,
   ติดตั้ง build dependency ของมันเอง (ซึ่งอาจมี dependency ของตัวเองอีกที เป็น chain ยาว)
   แล้ว configure/build/install ด้วยมือทุกขั้นตอน — งานที่ทำซ้ำแบบนี้ใน C++ ใช้เวลามากกว่า
   ภาษาอื่นอย่างเทียบกันไม่ได้

### ลองจินตนาการโลกที่ไม่มี Package Manager เลย

ก่อนจะมี vcpkg/Conan (และก่อนที่ distro จะมี `apt`/`brew` สมบูรณ์แบบอย่างทุกวันนี้) นักพัฒนา
C++ ต้อง build 3rd-party library ทุกตัวด้วยมือตามขั้นตอนคลาสสิกนี้ (อ้างอิงความรู้เรื่อง
Static/Dynamic Library จาก **Part 39**):

```bash
# 1. ดาวน์โหลด source code ของ library เอง (มักเป็น .tar.gz)
wget https://example.com/somelib-1.2.3.tar.gz
tar xzf somelib-1.2.3.tar.gz
cd somelib-1.2.3

# 2. อ่าน README เพื่อรู้ว่า build dependency ของ library นี้คืออะไรบ้าง
#    (บางที library ตัวนี้ก็ต้องพึ่ง library อื่นอีกทีเป็นทอดๆ)
./configure --prefix=/usr/local

# 3. Build จาก source เอง (ใช้เวลานานแค่ไหนขึ้นกับขนาด library)
make -j$(nproc)

# 4. ติดตั้งเข้าระบบด้วยสิทธิ์ root (มักเขียนทับ path ของระบบตรงๆ)
sudo make install

# 5. กลับไปที่โปรเจกต์ของเรา แล้วหวังว่า compiler จะหา header/library เจอ
g++ -I/usr/local/include -L/usr/local/lib myapp.cpp -lsomelib -o myapp
```

ถ้า library ที่ต้องการมี dependency ของตัวเองอีก 5-10 ตัว (เรื่องปกติมากสำหรับ library ขนาด
ใหญ่อย่าง OpenCV หรือ Boost) ก็ต้องทำ 5 ขั้นตอนนี้ซ้ำสำหรับทุก dependency ในลำดับที่ถูกต้อง
ด้วยมือ — และถ้าเพื่อนร่วมทีมอีกคนใช้ macOS ขั้นตอนเหล่านี้ก็อาจต้องปรับแก้ใหม่ทั้งหมด เพราะ
`./configure` บางตัวอาจไม่รองรับ macOS หรือ path ของระบบต่างกัน นี่คือปัญหาที่แท้จริงที่
vcpkg/Conan เกิดขึ้นมาแก้ไข: ให้กระบวนการทั้ง 5 ขั้นตอนนี้ (และการไล่ dependency graph ที่
ซับซ้อนกว่านี้มาก) ถูกทำให้อัตโนมัติ ทำซ้ำได้ และพอร์ตข้ามแพลตฟอร์มได้ในคำสั่งเดียว

เปรียบเทียบกับภาษาอื่นที่มี Package Manager มาตรฐานติดตัวมาตั้งแต่ต้น:

```bash
# JavaScript/Node.js — npm จัดการทุกอย่างให้ ข้ามแพลตฟอร์มได้เลย
npm install express

# Python — pip จัดการให้เช่นกัน
pip install requests
```

ทั้งสองคำสั่งนี้ทำงานเหมือนกันทุกประการไม่ว่าจะรันบน Windows, macOS, หรือ Linux เพราะภาษา
เหล่านี้ **ไม่ต้อง compile จาก source ทุกครั้ง** (ส่วนใหญ่แจกเป็น pre-built package หรือเป็น
Interpreted language) แต่ C++ เป็นภาษาที่ compile เป็น native machine code ซึ่งขึ้นกับ
compiler, C++ standard library implementation, และ ABI (Application Binary Interface) ของ
แต่ละแพลตฟอร์มอย่างมาก ทำให้การแจก pre-built binary ข้าม platform ทำได้ยากกว่ามาก
Package Manager สำหรับ C++ (vcpkg, Conan) จึงต้องแก้ปัญหาที่ซับซ้อนกว่านี้: ไม่ใช่แค่ดาวน์โหลด
ไฟล์ แต่ต้องรู้วิธี**คอมไพล์ library นั้นให้ตรงกับ compiler/platform/configuration ของเครื่อง
ผู้ใช้เองด้วย**

นี่คือเหตุผลที่วงการ C++ พัฒนา Package Manager เฉพาะทางขึ้นมา 2 ตัวหลักที่ได้รับความนิยม
สูงสุดในปัจจุบัน (2026): **vcpkg** และ **Conan** ซึ่งเราจะเรียนรู้ทั้งสองตัวใน Part นี้

> **หมายเหตุสำคัญเกี่ยวกับ Part นี้**: สภาพแวดล้อม sandbox ที่ใช้เขียนบทเรียนนี้ถูกจำกัดการ
> เชื่อมต่อ network ไปยัง server ของ vcpkg/Conan (จะพิสูจน์ให้เห็นเป็นรูปธรรมด้วย error จริง
> ในหัวข้อ 92.2 และ 92.3) ดังนั้นตัวอย่างในบทนี้จะ**ติดตั้งเครื่องมือทั้งสองตัวจริง** และแสดง
> **คำสั่ง/syntax/workflow ที่ถูกต้องตามหลักการจริง** แต่จะ**ไม่ได้ demo การดาวน์โหลด/build
> library จริงจนสำเร็จ** เพราะข้อจำกัดของ sandbox — ทุกจุดที่ล้มเหลวจะแสดง error message
> จริงพร้อมคำอธิบายว่าเกิดอะไรขึ้น แทนที่จะเสแสร้งว่าใช้งานได้สมบูรณ์

---

## 92.2 vcpkg: Package Manager จาก Microsoft (Step 730)

**vcpkg** เป็น Package Manager แบบ Open Source ที่ Microsoft เริ่มพัฒนาในปี 2016 เดิมเน้น
รองรับ Windows/Visual Studio เป็นหลัก แต่ปัจจุบันรองรับ Linux และ macOS อย่างสมบูรณ์เช่นกัน
แนวคิดหลักของ vcpkg คือระบบ **Port-based**: แต่ละ library ที่ vcpkg รองรับจะมีไฟล์ที่เรียกว่า
**Port** (สคริปต์ CMake ที่บอกวิธี download source, configure, build, และ install library นั้น)
เก็บอยู่ใน repository กลางที่ Microsoft และชุมชนช่วยกันดูแล

### ติดตั้ง vcpkg จริง

vcpkg แจกจ่ายเป็น source code ให้ clone มา bootstrap เอง (ไม่มีแจกผ่าน `apt`):

```bash
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg
./bootstrap-vcpkg.sh -disableMetrics
```

ผลลัพธ์จริงที่ได้บนเครื่องนี้:

```
Downloading vcpkg-glibc...
vcpkg package management program version 2026-07-27-98d7cb0cf1f4686a3e43aa5672b6230c1d56bce8

See LICENSE.txt for license information.
```

การ bootstrap สำเร็จจริง — ได้ executable ชื่อ `vcpkg` มาใช้งาน ตรวจสอบเวอร์ชัน:

```bash
$ ./vcpkg version
vcpkg package management program version 2026-07-27-98d7cb0cf1f4686a3e43aa5672b6230c1d56bce8

See LICENSE.txt for license information.
```

### แนวคิด Triplet — กำหนดว่า "build เพื่อแพลตฟอร์ม/สถาปัตยกรรมไหน"

vcpkg ใช้คำว่า **Triplet** อธิบายเป้าหมายของการ build เช่น `x64-linux` (สถาปัตยกรรม 64-bit
บน Linux), `x64-windows` (64-bit บน Windows แบบ Dynamic Library), `x64-windows-static`
(64-bit บน Windows แบบ Static Library) — สิ่งนี้สำคัญมากสำหรับ C++ เพราะ ABI ของแต่ละ
แพลตฟอร์ม/configuration แตกต่างกัน ต้องระบุให้ชัดเจนว่าต้องการ build แบบไหน

### ทดลองติดตั้ง Library จริงด้วย vcpkg

ลองสั่งติดตั้ง library `fmt` (ไลบรารี string formatting ที่ได้รับความนิยมสูงมาก และเป็นต้นแบบ
ของ `std::format` ใน C++20):

```bash
./vcpkg install fmt
```

ผลลัพธ์จริงในช่วงแรก — vcpkg เริ่มทำงาน **สำเร็จบางส่วน**:

```
Detecting compiler hash for triplet x64-linux...
Compiler found: /usr/bin/c++
Restored 0 package(s) from /root/.cache/vcpkg/archives in 16.9 us.
Installing 1/3 vcpkg-cmake:x64-linux@2025-08-07...
Building vcpkg-cmake:x64-linux@2025-08-07...
-- Installing: .../share/vcpkg-cmake/vcpkg_cmake_configure.cmake
-- Installing: .../share/vcpkg-cmake/vcpkg_cmake_build.cmake
-- Installing: .../share/vcpkg-cmake/vcpkg_cmake_install.cmake
-- Installing: .../share/vcpkg-cmake/vcpkg-port-config.cmake
-- Installing: .../share/vcpkg-cmake/copyright
-- Performing post-build validation
Elapsed time to handle vcpkg-cmake:x64-linux: 19 ms
Installing 2/3 vcpkg-cmake-config:x64-linux@2026-07-21...
Building vcpkg-cmake-config:x64-linux@2026-07-21...
-- Installing: .../share/vcpkg-cmake-config/vcpkg_cmake_config_fixup.cmake
-- Installing: .../share/vcpkg-cmake-config/vcpkg-port-config.cmake
-- Skipping post-build validation due to VCPKG_POLICY_EMPTY_PACKAGE
Elapsed time to handle vcpkg-cmake-config:x64-linux: 18.5 ms
Installing 3/3 fmt:x64-linux@12.2.0#1...
Building fmt:x64-linux@12.2.0#1...
Downloading https://github.com/fmtlib/fmt/commit/588b3a0f8f6a8bcf2a959cae882d5b2703e86737.patch?full_index=1 -> fmt-backport-4813.patch
error: curl operation failed with response code 403.
error: Not a transient network error, won't retry download from https://github.com/fmtlib/fmt/commit/588b3a0f8f6a8bcf2a959cae882d5b2703e86737.patch?full_index=1
note: If you are using a proxy, please ensure your proxy settings are correct.

CMake Error at scripts/cmake/vcpkg_download_distfile.cmake:134 (message):
  Download failed, halting portfile.

error: building fmt:x64-linux failed with: BUILD_FAILED
```

นี่คือผลลัพธ์**จริง**ที่เกิดขึ้นในสภาพแวดล้อมของบทเรียนนี้ ควรค่าแก่การวิเคราะห์อย่างละเอียด:

- **`vcpkg-cmake` และ `vcpkg-cmake-config`** (สอง helper port ที่ vcpkg ใช้ภายในสำหรับ
  integrate กับ CMake) **build สำเร็จ** เพราะ source ของสอง port นี้ไม่ต้องดาวน์โหลดไฟล์
  เพิ่มเติมจากภายนอกอีก
- **`fmt` build ล้มเหลว** เพราะ port ของ `fmt` ต้องดาวน์โหลด patch file เพิ่มเติมจาก GitHub
  แต่ proxy ของ sandbox ปฏิเสธการเชื่อมต่อ (HTTP 403 Forbidden) — นี่ไม่ใช่บั๊กของ vcpkg
  แต่เป็นข้อจำกัดของสภาพแวดล้อมที่ใช้รันบทเรียนนี้โดยเฉพาะ

> **บทเรียนสำคัญจากความล้มเหลวนี้**: นี่คือตัวอย่างจริงของปัญหาที่ทีมวิศวกรรมมืออาชีพเจอบ่อย
> มากใน CI/CD Pipeline ที่รันในสภาพแวดล้อมปิด (Sandboxed/Air-gapped Environment) — ถ้า
> Build Agent ไม่มีสิทธิ์เข้าถึง internet เต็มรูปแบบ (ด้วยเหตุผลด้าน Security) การติดตั้ง
> Package ผ่าน vcpkg/Conan แบบดึงจาก internet สดๆ ทุกครั้งจะล้มเหลวเสมอ วิธีแก้ในโลกจริง
> คือการตั้ง **Binary Cache** ภายในองค์กร (Private Registry) ให้ CI ดึงจากภายในแทนการยิงออก
> internet ตรงๆ ทุกครั้ง — เป็นหัวข้อที่ทีม DevOps/Platform Engineering ต้องออกแบบให้ดี

### ค้นหา Port ที่มีอยู่แบบ Offline ด้วย vcpkg search

แม้จะดาวน์โหลด library จริงไม่สำเร็จเพราะข้อจำกัดของ network แต่ vcpkg เก็บ **รายชื่อ Port
ทั้งหมด** ไว้เป็นไฟล์ในเครื่องตั้งแต่ตอน `git clone` แล้ว (เพราะ Port คือสคริปต์ที่อยู่ใน
repository ของ vcpkg เอง) คำสั่ง `search` จึงทำงานแบบ **Offline ได้เต็มรูปแบบ** โดยไม่ต้อง
ติดต่อ network เลย:

```bash
./vcpkg search fmt
```

ผลลัพธ์จริง:

```
fmt                      12.2.0#1         {fmt} is an open-source formatting library providing a fast and safe alter...
log4cxx[fmt]                              Include the log4cxx::FMTLayout class that uses libfmt to layout messages
loguru[fmt]                               Build with fmt support in non-header-only mode
rivers[fmt]                               Use fmt as rivers fommatter
serdepp                  0.1.4.1          c++ 17 universal serialize deserialize library like rust serde, support li...
spdlog[fmt]                               Use fmt library
The result may be outdated. Run `git pull` to get the latest results.
```

สังเกตว่า vcpkg บอกเวอร์ชันล่าสุดของ `fmt` ที่มีอยู่ในระบบ Port ตอนนี้คือ `12.2.0` ได้ถูกต้อง
แม้ไม่มี network เลย เพราะข้อมูลนี้เป็นแค่การอ่านไฟล์ metadata ในเครื่อง ไม่ใช่การติดต่อ server
ภายนอก — ส่วนที่ต้องใช้ network จริงๆ มีแค่ตอน**ดาวน์โหลด source code ของ library** มา build
เท่านั้น (ขั้นตอนที่ล้มเหลวในหัวข้อก่อนหน้า)

### Syntax และ Workflow มาตรฐานของ vcpkg (สำหรับใช้งานจริงเมื่อมี network)

แม้จะ demo การติดตั้งจริงให้เสร็จสมบูรณ์ไม่ได้ในบทเรียนนี้ แต่ syntax และ workflow ต่อไปนี้
คือของจริงที่ใช้ในโปรเจกต์ทั่วโลก (อ้างอิงจาก documentation ทางการของ vcpkg):

**โหมด Classic** (ติดตั้ง package ไปที่ vcpkg เอง ใช้ร่วมกันได้หลายโปรเจกต์):

```bash
./vcpkg install fmt nlohmann-json
```

**โหมด Manifest** (แนะนำสำหรับโปรเจกต์จริง — ประกาศ dependency ไว้ในไฟล์ `vcpkg.json`
ที่อยู่ใน repository ของโปรเจกต์เอง เหมือน `package.json` ของ npm):

```json
{
  "name": "my-cpp-project",
  "version": "1.0.0",
  "dependencies": [
    "fmt",
    "nlohmann-json",
    { "name": "zlib", "version>=": "1.3.0" }
  ]
}
```

เมื่อมีไฟล์ `vcpkg.json` แล้ว แค่สั่ง configure CMake ตามปกติ vcpkg ก็จะติดตั้ง dependency
ทั้งหมดในไฟล์นี้ให้อัตโนมัติ (Manifest Mode)

### เชื่อมต่อกับ CMake ผ่าน Toolchain File

นี่คือจุดเชื่อมสำคัญที่สุด: vcpkg ทำงานร่วมกับ CMake ผ่าน **Toolchain File** ที่ vcpkg สร้างให้
เอง เพียงส่ง path ของไฟล์นี้ตอน configure:

```bash
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=/path/to/vcpkg/scripts/buildsystems/vcpkg.cmake
```

หลังจากนั้นใน `CMakeLists.txt` เขียน `find_package` ตามปกติเหมือนกับ library ที่ติดตั้งผ่าน
`apt` ทุกประการ — **นี่คือจุดที่สวยงามที่สุดของการออกแบบระบบนี้**: โค้ด CMake ไม่จำเป็นต้องรู้
เลยว่า library มาจาก vcpkg, Conan, หรือ system package เพราะ `find_package` เป็น interface
เดียวกันหมด:

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyApp LANGUAGES CXX)

find_package(fmt CONFIG REQUIRED)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE fmt::fmt)
```

`fmt::fmt` คือชื่อ **Imported Target** ที่ package ประกาศไว้ให้ (แนวทางเดียวกับ `ZLIB::ZLIB`
ที่จะเห็นในหัวข้อ 92.5) — ผู้ใช้ไม่ต้องรู้เองว่า include path หรือ library path จริงๆ อยู่ที่ไหน
เพราะ `find_package` + `target_link_libraries` จัดการให้หมดผ่านกลไก `PUBLIC`/`INTERFACE`
ที่เรียนใน Part 91

---

## 92.3 Conan: Package Manager แบบ Cross-Platform (Step 731)

**Conan** เป็น Package Manager สำหรับ C/C++ ที่พัฒนาโดยบริษัท JFrog เน้นการแก้ปัญหา
**Dependency Resolution** อย่างจริงจัง (คล้ายกับที่ `pip`/`npm` ทำในภาษาอื่น) — จุดต่างสำคัญ
จาก vcpkg คือ Conan ไม่ผูกกับ Microsoft และออกแบบมาให้ตัดสินใจเรื่อง binary compatibility
(ว่า pre-built binary ที่มีอยู่แล้วใน cache ตรงกับ compiler/settings ปัจจุบันหรือไม่) ได้อย่าง
ละเอียดผ่านระบบที่เรียกว่า **Profile**

### ติดตั้งและตรวจสอบ Conan

Conan เป็นโปรแกรม Python ติดตั้งผ่าน `pip`:

```bash
pip install conan
conan --version
```

บนเครื่องที่ใช้เขียนบทเรียนนี้มี Conan ติดตั้งไว้แล้ว:

```
$ conan --version
Conan version 2.27.0
```

### Profile — บอก Conan ว่าเครื่องนี้มี Compiler/Settings อะไรบ้าง

ก่อนใช้งาน Conan ต้องสร้าง **Profile** ที่อธิบายสภาพแวดล้อมของเครื่อง (compiler, เวอร์ชัน,
C++ standard library, สถาปัตยกรรม) คำสั่งนี้ตรวจจับให้อัตโนมัติ:

```bash
conan profile detect --force
```

ผลลัพธ์จริงบนเครื่องนี้:

```
detect_api: Found cc=gcc-13.3.0
detect_api: gcc>=5, using the major as version
detect_api: gcc C++ standard library: libstdc++11

Detected profile:
[settings]
arch=x86_64
build_type=Release
compiler=gcc
compiler.cppstd=gnu17
compiler.libcxx=libstdc++11
compiler.version=13
os=Linux

WARN: This profile is a guess of your environment, please check it.
WARN: The output of this command is not guaranteed to be stable and can change in future Conan versions.
WARN: Use your own profile files for stability.
Saving detected profile to /root/.conan2/profiles/default
```

Conan ตรวจพบ GCC 13, C++ standard library เป็น `libstdc++11` (GNU libstdc++ เวอร์ชัน C++11
ABI ขึ้นไป — ตรงกับที่เรียนเรื่อง ABI compatibility ใน Part 39) และสถาปัตยกรรม `x86_64` ได้
ถูกต้องทั้งหมด — Profile นี้สำคัญมากเพราะ Conan ใช้ข้อมูลนี้ตัดสินใจว่า **binary ที่มีอยู่แล้วใน
cache กลาง (Conan Center) ตรงกับสภาพแวดล้อมของเราหรือไม่** ถ้าตรงกันก็ดาวน์โหลด binary
สำเร็จรูปมาใช้เลยโดยไม่ต้อง compile เอง (ประหยัดเวลามาก) ถ้าไม่ตรงต้อง build จาก source

### ทดลองติดตั้ง Library จริงด้วย conanfile.txt

สร้างไฟล์ `conanfile.txt` ประกาศ dependency (รูปแบบไฟล์มาตรฐานของ Conan, คล้ายกับ
`vcpkg.json`):

```ini
[requires]
zlib/1.3.1

[generators]
CMakeDeps
CMakeToolchain
```

รันคำสั่งติดตั้ง:

```bash
conan install . --output-folder=build --build=missing
```

ผลลัพธ์จริงในสภาพแวดล้อมของบทเรียนนี้:

```
======== Input profiles ========
Profile host:
[settings]
arch=x86_64
build_type=Release
compiler=gcc
compiler.cppstd=gnu17
compiler.libcxx=libstdc++11
compiler.version=13
os=Linux

Profile build:
[settings]
arch=x86_64
build_type=Release
compiler=gcc
compiler.cppstd=gnu17
compiler.libcxx=libstdc++11
compiler.version=13
os=Linux


======== Computing dependency graph ========
zlib/1.3.1: Not found in local cache, looking in remotes...
zlib/1.3.1: Checking remote: conancenter
Connecting to remote 'conancenter' anonymously
Graph root
    conanfile.txt: /home/user/conan_test/conanfile.txt
ERROR: Package 'zlib/1.3.1' not resolved: HTTPSConnectionPool(host='center2.conan.io', port=443):
Max retries exceeded with url: /v1/ping (Caused by ProxyError('Unable to connect to proxy',
OSError('Tunnel connection failed: 403 Forbidden')))

Unable to connect to remote conancenter=https://center2.conan.io
1. Make sure the remote is reachable or,
2. Disable it with 'conan remote disable <remote>' or,
3. Use the '-nr/--no-remote' argument
Then try again.. Required by 'cli'
```

เหมือนกับกรณีของ vcpkg ทุกประการ: Conan **ทำงานถูกต้องตามที่ควรจะเป็น** (ตรวจสอบ profile,
คำนวณ dependency graph, พยายามติดต่อ Conan Center) แต่ **ล้มเหลวที่ขั้นตอนติดต่อ network**
ด้วย error เดียวกัน (`403 Forbidden` จาก proxy ของ sandbox) — ยืนยันอีกครั้งว่านี่เป็นข้อจำกัด
ของสภาพแวดล้อมที่ใช้รันบทเรียน ไม่ใช่ปัญหาของตัวเครื่องมือ

### Syntax และ Workflow มาตรฐานของ Conan (สำหรับใช้งานจริงเมื่อมี network)

**conanfile.txt** (แบบง่าย เหมาะกับผู้เริ่มต้น เขียนแค่ list ของ dependency):

```ini
[requires]
fmt/11.0.2
nlohmann_json/3.11.3

[generators]
CMakeDeps
CMakeToolchain

[layout]
cmake_layout
```

**conanfile.py** (แบบยืดหยุ่นกว่า เหมาะกับโปรเจกต์ที่ซับซ้อน เพราะเป็น Python เต็มรูปแบบ
สามารถเขียน logic เงื่อนไขได้):

```python
from conan import ConanFile
from conan.tools.cmake import CMakeToolchain, CMakeDeps, cmake_layout

class MyAppRecipe(ConanFile):
    settings = "os", "compiler", "build_type", "arch"
    generators = "CMakeDeps", "CMakeToolchain"

    def requirements(self):
        self.requires("fmt/11.0.2")
        self.requires("nlohmann_json/3.11.3")

    def layout(self):
        cmake_layout(self)
```

Workflow มาตรฐานทั้งหมด (เมื่อมี network เชื่อมต่อ Conan Center ได้ปกติ):

```bash
# 1. ติดตั้ง dependency ทั้งหมด (ดาวน์โหลด pre-built binary ถ้ามี หรือ build จาก source ถ้าไม่มี)
conan install . --output-folder=build --build=missing

# 2. configure CMake โดยใช้ toolchain file ที่ Conan generate ให้
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake

# 3. build ตามปกติ
cmake --build build
```

และใน `CMakeLists.txt` เขียน `find_package` เหมือนเดิมทุกประการ (นี่คือจุดที่ Conan กับ vcpkg
เหมือนกัน — ทั้งคู่ทำให้ `find_package` ทำงานได้โดยไม่ต้องติดตั้ง library เข้าระบบจริงๆ):

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyApp LANGUAGES CXX)

find_package(fmt REQUIRED)
find_package(nlohmann_json REQUIRED)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE fmt::fmt nlohmann_json::nlohmann_json)
```

### ตรวจสอบ Local Cache ของ Conan แบบ Offline

เช่นเดียวกับ vcpkg, Conan ก็มีคำสั่งที่ทำงานแบบ Offline ได้ — `conan list` ใช้ตรวจสอบว่า
เครื่องนี้มี package ตัวไหนอยู่ใน **Local Cache** แล้วบ้าง (ไม่ต้องพึ่ง remote):

```bash
conan list "*"
```

ผลลัพธ์จริงบนเครื่องนี้ (เพราะยังไม่เคย build/download package ใดสำเร็จเลย):

```
Local Cache
  WARN: There are no matching recipe references
```

ผลลัพธ์นี้ยืนยันตรงไปตรงมาว่า Local Cache ว่างเปล่า 100% ซึ่งสอดคล้องกับที่เราพิสูจน์ไปแล้วว่า
`conan install` ไม่สามารถดาวน์โหลด `zlib` มาสำเร็จได้ในสภาพแวดล้อมนี้ — ถ้าใน future มี
package ถูกดาวน์โหลด/build สำเร็จแล้ว คำสั่งนี้จะแสดงรายชื่อ package พร้อมเวอร์ชันและ
Package ID (hash ที่คำนวณจาก settings ของ profile) ที่เก็บไว้ในเครื่อง

### Lockfile: อีกชั้นของ Reproducibility ที่ทั้งสองเครื่องมือมีให้

ทั้ง vcpkg และ Conan มีกลไก **Lockfile** ที่ล็อกไม่ใช่แค่เวอร์ชันที่ระบุใน manifest แต่ล็อกถึง
**เวอร์ชันที่ resolve ได้จริงของทุก dependency ของ dependency** (transitive dependency) ทำให้
การ build ครั้งถัดไปได้ผลลัพธ์เหมือนเดิมเป๊ะแม้เวลาจะผ่านไปนานแค่ไหนก็ตาม (คล้ายกับ
`package-lock.json` ของ npm หรือ `Cargo.lock` ของ Rust):

```bash
# Conan: สร้าง lockfile จาก dependency graph ปัจจุบัน
conan lock create conanfile.txt --lockfile-out=conan.lock

# ใช้ lockfile ตอน install ครั้งถัดไป เพื่อบังคับให้ได้ผลลัพธ์เหมือนเดิมเป๊ะ
conan install . --lockfile=conan.lock
```

```bash
# vcpkg: ไฟล์ vcpkg.json ที่ระบุ builtin-baseline คือกลไกล็อก baseline ของ Port ทั้งหมด
{
  "name": "my-cpp-project",
  "version": "1.0.0",
  "builtin-baseline": "a1b2c3d4e5f6...",
  "dependencies": ["fmt"]
}
```

`builtin-baseline` ของ vcpkg คือ commit hash ของ vcpkg ports repository ที่ต้องการ "หยุดเวลา"
ไว้ ทำให้แม้ vcpkg ports จะมีการอัปเดตเวอร์ชันของ `fmt` ในอนาคต โปรเจกต์นี้ก็ยังคงได้เวอร์ชันที่
ตรงกับตอนที่ commit hash นี้ถูกบันทึกไว้เสมอ — ควร commit ทั้ง `vcpkg.json`/`conan.lock` เข้า
Git ควบคู่กับ source code เสมอ เพื่อให้ Reproducibility สมบูรณ์แบบจริงๆ

---

## 92.4 เปรียบเทียบ vcpkg กับ Conan (Step 732)

| หัวข้อ | vcpkg | Conan |
|---|---|---|
| **ผู้พัฒนา** | Microsoft (Open Source) | JFrog (Open Source) |
| **แนวคิดหลัก** | Port-based: สคริปต์ CMake สร้าง/build library จาก source | Package + Binary Cache: เน้นดาวน์โหลด pre-built binary ที่ตรงกับ profile ก่อนเสมอ |
| **ไฟล์ประกาศ dependency** | `vcpkg.json` (Manifest Mode) | `conanfile.txt` หรือ `conanfile.py` |
| **ภาษาที่ใช้เขียน Recipe** | CMake script (`portfile.cmake`) | Python (`conanfile.py`) — ยืดหยุ่นกว่ามากสำหรับ logic ซับซ้อน |
| **การเชื่อมกับ CMake** | Toolchain File (`vcpkg.cmake`) | Toolchain File + `CMakeDeps`/`CMakeToolchain` generator |
| **จุดแข็งดั้งเดิม** | รองรับ Windows/Visual Studio ได้ลึกมาก (เกิดมาเพื่อสิ่งนี้) | จัดการ Binary Compatibility และ Dependency Graph ที่ซับซ้อนได้ดีมาก (หลาย version ของ lib เดียวกันพร้อมกัน) |
| **Binary Cache กลาง** | มี (Microsoft Artifact/registry ที่ทีมตั้งเองได้) | มี (JFrog Artifactory, ConanCenter) |
| **Community/Registry สาธารณะ** | vcpkg ports (repository เดียว บน GitHub) | ConanCenter |
| **ความนิยมในวงการเกม/Windows** | สูงมาก | สูงเช่นกัน แต่เดิมนิยมในสาย Embedded/Cross-compilation มากกว่า |
| **License/ค่าใช้จ่าย** | ฟรี, Open Source ทั้งหมด | ฟรีสำหรับใช้งานพื้นฐาน (Artifactory แบบ Enterprise มีค่าใช้จ่าย) |

**เมื่อไหร่ควรเลือกอะไร**:

- ถ้าโปรเจกต์เน้น Windows/Visual Studio เป็นหลัก หรือทีมคุ้นเคยกับ ecosystem ของ Microsoft
  อยู่แล้ว **vcpkg** มักเป็นตัวเลือกที่ลื่นไหลกว่าเพราะ integrate เข้ากับ Visual Studio ได้ลึกมาก
- ถ้าโปรเจกต์ต้อง cross-compile ไปหลาย platform พร้อมกัน (เช่น งาน Embedded ที่ต้อง build
  เดียวกันให้ทั้ง ARM และ x86) หรือต้องจัดการ dependency graph ที่ซับซ้อนมาก (หลาย library
  ที่ต้องการเวอร์ชันของ dependency ร่วมกันต่างกัน) **Conan** มักมีเครื่องมือรองรับที่ยืดหยุ่นกว่า
- ทั้งสองตัวรองรับ Linux/macOS/Windows ได้ครบเหมือนกันในปัจจุบัน (2026) ความต่างที่แท้จริง
  จึงมักขึ้นกับ**ความคุ้นเคยของทีม**และ**ecosystem ของ library ที่ต้องการ**มากกว่าความสามารถ
  พื้นฐาน — ในหลายบริษัทถึงกับใช้ทั้งสองตัวคู่กันในโปรเจกต์ต่างกัน

---

## 92.5 System Package เทียบกับ Package Manager สมัยใหม่ (Step 733)

ตลอดหลักสูตรนี้ (ตั้งแต่ `libbenchmark-dev` ใน Part 90 ไปจนถึง `libgtest-dev` ที่จะใช้ใน
Part 93) เราใช้แนวทาง **System Package** ผ่าน `apt install` มาตลอด เพื่อให้เห็นความต่างอย่าง
เป็นรูปธรรม เราจะทดลอง**จริง**กับ library ที่มีอยู่แล้วในระบบผ่าน `apt`: `zlib1g-dev`
(ไลบรารีบีบอัดข้อมูลที่ใช้กันแพร่หลายมาก)

### ทดสอบจริง: find_package(ZLIB) ผ่าน System Package

```bash
sudo apt install zlib1g-dev -y
```

โปรเจกต์ทดสอบ `main.cpp`:

```cpp
#include <zlib.h>
#include <cstdio>

int main() {
    printf("zlib version linked at runtime: %s\n", zlibVersion());
    return 0;
}
```

`CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.20)
project(ZlibDemo LANGUAGES CXX)

find_package(ZLIB REQUIRED)

add_executable(zlib_demo main.cpp)
target_link_libraries(zlib_demo PRIVATE ZLIB::ZLIB)
```

Configure และ Build จริง:

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
-- Found ZLIB: /usr/lib/x86_64-linux-gnu/libz.so (found version "1.3")
-- Configuring done (0.2s)
-- Generating done (0.0s)
-- Build files have been written to: /home/user/zlib_demo/build
```

```bash
cmake --build .
./zlib_demo
```

```
[ 50%] Building CXX object CMakeFiles/zlib_demo.dir/main.cpp.o
[100%] Linking CXX executable zlib_demo
[100%] Built target zlib_demo
zlib version linked at runtime: 1.3
```

**นี่คือจุดสำคัญที่สุดของหัวข้อนี้**: `find_package(ZLIB REQUIRED)` กับ `target_link_libraries(...
ZLIB::ZLIB)` เป็น**โค้ด CMake ชุดเดียวกันเป๊ะๆ**ไม่ว่า `zlib` จะมาจาก `apt`, จาก vcpkg, หรือ
จาก Conan — CMake มีระบบ **Find Module** (`FindZLIB.cmake` ที่มากับ CMake เอง) ที่รู้วิธี
ค้นหา zlib จาก path มาตรฐานของระบบให้อัตโนมัติ ถ้าเปลี่ยนไปใช้ vcpkg/Conan สิ่งที่เปลี่ยนคือ
**แหล่งที่มาของไฟล์ library** (จาก `/usr/lib/...` เป็น path ภายใน vcpkg/Conan cache) แต่โค้ด
`CMakeLists.txt` ที่ผู้ใช้ library เขียนแทบไม่ต้องแก้ไขอะไรเลย

### ตารางเปรียบเทียบ System Package vs vcpkg vs Conan

| ประเด็น | System Package (`apt`) | vcpkg | Conan |
|---|---|---|---|
| **Portability ข้าม OS** | ต่ำมาก — ชื่อ/วิธีติดตั้งต่างกันทุก distro/OS | สูง — คำสั่งเดียวกันทำงานบน Linux/macOS/Windows | สูง — เหมือน vcpkg |
| **Version Pinning ต่อโปรเจกต์** | ทำไม่ได้ (ผูกกับเวอร์ชันที่ OS แจกมาให้) | ทำได้ผ่าน `vcpkg.json` (ระบุ version constraint ได้) | ทำได้ผ่าน `conanfile` (ระบุเวอร์ชันแม่นยำ เช่น `zlib/1.3.1`) |
| **Reproducible Build ข้ามเครื่อง/ข้ามเวลา** | ต่ำ — ขึ้นกับว่าเครื่องนั้น update `apt` ล่าสุดเมื่อไหร่ | สูง — ล็อกเวอร์ชันไว้ในไฟล์ manifest commit เข้า Git ได้ | สูง — เช่นเดียวกัน |
| **ความเร็วตอน setup ครั้งแรก** | เร็วมาก (มี binary พร้อมใช้ในตัว) | ช้ากว่า (อาจต้อง build จาก source ถ้าไม่มี binary cache) | ช้ากว่า เว้นแต่มี binary ตรงกับ profile ใน cache |
| **สิทธิ์ผู้ดูแลระบบ (root/sudo)** | ต้องการ (ติดตั้งระดับระบบ) | ไม่ต้องการ (ติดตั้งแยกต่อโปรเจกต์ในโฟลเดอร์ของตัวเอง) | ไม่ต้องการ เช่นเดียวกัน |
| **เหมาะกับ** | เครื่องมือพื้นฐานที่ใช้ร่วมกันทั้งเครื่อง, CI ที่ควบคุม base image เองได้ | โปรเจกต์ C++ ที่ต้อง portable ข้าม OS จริงจัง โดยเฉพาะทีมที่ใช้ Visual Studio | โปรเจกต์ขนาดใหญ่ที่มี dependency graph ซับซ้อน หรือต้อง cross-compile |

**ประเด็นเรื่อง Reproducibility คือหัวใจสำคัญที่สุด**: สมมติทีมมี 2 คน คนแรกใช้ Ubuntu 22.04
คนที่สองใช้ Ubuntu 24.04 ถ้าทั้งคู่ `apt install libfmt-dev` อาจได้ `fmt` คนละเวอร์ชันกันโดย
สิ้นเชิง (Ubuntu แต่ละเวอร์ชันแจก package version ต่างกัน) โค้ดที่ compile ผ่านในเครื่องคนแรก
อาจ compile ไม่ผ่านในเครื่องคนที่สองถ้ามีการใช้ API ที่เพิ่งเพิ่มมาในเวอร์ชันใหม่กว่า — ในขณะที่
ถ้าใช้ vcpkg/Conan พร้อมไฟล์ manifest ที่ commit เข้า Git ทั้งสองคนจะได้ `fmt` เวอร์ชัน
**เดียวกันเป๊ะ**ไม่ว่าจะรันบน distro ไหนก็ตาม เพราะเวอร์ชันถูกระบุไว้ในไฟล์ manifest ไม่ใช่ขึ้นกับ
ว่า OS แจก package เวอร์ชันอะไรมาให้

---

## 92.6 เลือกแนวทางที่เหมาะสมกับงานจริง (Step 734–736)

### Step 734: เมื่อไหร่ควรใช้ System Package

- โปรเจกต์เป็น Internal Tool ที่รันบนเครื่อง/Docker image ที่ทีมควบคุม base image เองอยู่แล้ว
  (กำหนดเวอร์ชัน OS ตายตัว ทุกคนในทีมใช้ image เดียวกัน) — ในกรณีนี้ปัญหาเรื่อง version
  ต่างกันข้ามเครื่องแทบไม่มี เพราะทุกคน "เครื่องเดียวกัน" ในทางปฏิบัติอยู่แล้ว
- Library ที่เป็นส่วนหนึ่งของระบบปฏิบัติการอยู่แล้วจริงๆ เช่น `pthread` (POSIX Threads จาก
  Part 31), `libc`, `libm` — ไลบรารีเหล่านี้ผูกกับ OS โดยธรรมชาติ ไม่มีเหตุผลต้องผ่าน Package
  Manager แยกต่างหาก
- ต้องการความเร็วในการ setup CI สูงสุด และ base image ของ CI มี package ที่ต้องการอยู่แล้ว

### Step 735: เมื่อไหร่ควรใช้ vcpkg/Conan

- โปรเจกต์ Open Source ที่ต้องรองรับผู้ใช้จากหลาย OS (Windows/macOS/Linux) พร้อมกัน และ
  ไม่สามารถบังคับให้ผู้ใช้ทุกคน `apt install` แบบเดียวกันได้ (โดยเฉพาะผู้ใช้ Windows ที่ไม่มี
  `apt` เลย)
- ต้องการ Version Pinning ที่แม่นยำ เพื่อ Reproducible Build ข้ามเครื่อง/ข้ามเวลา (สำคัญมาก
  สำหรับโปรเจกต์ที่มีอายุยาวและทีมงานเปลี่ยนคนไปเรื่อยๆ)
- ต้องการ library เวอร์ชันที่ใหม่กว่าหรือเก่ากว่าที่ OS แจกให้ หรือ library ที่ไม่มีใน apt
  repository เลย

### Step 736: Workflow แบบผสมที่พบบ่อยในทีมมืออาชีพจริง

ในทางปฏิบัติ หลายทีมไม่ได้เลือกใช้แนวทางใดแนวทางหนึ่งแบบเด็ดขาด แต่ผสมทั้งสองแบบตาม
ความเหมาะสม:

```cmake
cmake_minimum_required(VERSION 3.20)
project(MixedApp LANGUAGES CXX)

# pthread: มากับระบบปฏิบัติการอยู่แล้ว ใช้ system-level ตรงๆ
find_package(Threads REQUIRED)

# fmt: ต้องการเวอร์ชันเฉพาะเจาะจงและ portable ข้าม OS ให้ผู้ร่วมโครงการ Windows ใช้ได้ด้วย
# → มาจาก vcpkg/Conan ผ่าน find_package เหมือนกัน แค่ configure ด้วย toolchain file
find_package(fmt REQUIRED)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE Threads::Threads fmt::fmt)
```

สังเกตว่าโค้ด CMake **ไม่รู้และไม่จำเป็นต้องรู้เลย**ว่า `Threads` มาจาก system ส่วน `fmt` มาจาก
Package Manager ภายนอก — ทั้งหมดถูกซ่อนอยู่หลัง interface เดียวกันคือ `find_package` +
`target_link_libraries` ทุกประการ นี่คือพลังของการออกแบบ CMake ที่ทำให้ **การเปลี่ยนแหล่งที่มา
ของ dependency ไม่กระทบโค้ดที่ใช้งานมันเลย** ตราบใดที่ library นั้นมี CMake Config File หรือ
Find Module ที่ถูกต้อง

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมส่ง `CMAKE_TOOLCHAIN_FILE` ตอน configure** — ถ้าติดตั้ง library ผ่าน vcpkg/Conan
   ไว้แล้วแต่ configure CMake โดยไม่ระบุ toolchain file ที่ package manager สร้างให้
   `find_package` จะหา library นั้นไม่เจอเลย (มองไม่เห็น path ที่ vcpkg/Conan ติดตั้งไว้) เพราะ
   library เหล่านี้ไม่ได้ถูกติดตั้งไว้ที่ path มาตรฐานของระบบ (`/usr/lib`) เหมือน system package
2. **ผสม library จาก system package กับ vcpkg/Conan ในเวอร์ชันที่ขัดแย้งกัน** — เช่น
   `apt install libssl-dev` ไว้แล้ว แต่โปรเจกต์ดันประกาศ `openssl` ผ่าน vcpkg ด้วย อาจได้
   library 2 เวอร์ชันปนกันจนเกิด ODR Violation (One Definition Rule) หรือ linker error ที่
   สับสนมาก ควรเลือกแหล่งที่มาเดียวต่อ library หนึ่งตัวเสมอในโปรเจกต์เดียว
3. **ไม่ commit ไฟล์ manifest (`vcpkg.json`/`conanfile.txt`) เข้า Git** — ทำให้เพื่อนร่วมทีม
   หรือ CI ไม่รู้ว่าโปรเจกต์นี้ต้องการ dependency อะไรบ้าง สูญเสียประโยชน์เรื่อง Reproducible
   Build ไปทั้งหมด (ควร commit ไฟล์ manifest เสมอ เหมือนที่ commit `package-lock.json` ใน
   โปรเจกต์ Node.js)
4. **สมมติว่า CI/Build Agent มี internet access เต็มรูปแบบเสมอ** — ดังที่พิสูจน์ให้เห็นจริงใน
   หัวข้อ 92.2 และ 92.3 สภาพแวดล้อมปิด (Sandboxed CI, Air-gapped Network) เป็นเรื่องปกติมาก
   ในองค์กรที่ให้ความสำคัญกับ Security ควรวางแผนเรื่อง Binary Cache ภายในองค์กรตั้งแต่แรก
   ไม่ใช่คิดทีหลังตอนที่ CI ล้มเหลวกะทันหัน
5. **ไม่ล็อกเวอร์ชันให้แม่นยำ (ใช้ range แบบกว้างเกินไป)** — เช่นเขียน `fmt/*` แทนที่จะเป็น
   `fmt/11.0.2` ทำให้แต่ละครั้งที่ build อาจได้ library คนละเวอร์ชันกัน (ถ้ามีเวอร์ชันใหม่ออกมา
   ระหว่างทาง) สูญเสียจุดประสงค์หลักของการใช้ Package Manager ไปโดยสิ้นเชิง
6. **ไม่ตรวจสอบ License ของ library ที่ดึงมาผ่าน Package Manager** — เพราะการติดตั้งง่าย
   เกินไปทำให้บางทีมลืมตรวจสอบว่า license ของ library ที่เพิ่มเข้ามาเข้ากันได้กับ license ของ
   โปรเจกต์ตัวเองหรือไม่ (โดยเฉพาะเมื่อ deploy เป็น commercial product) เป็นความรับผิดชอบที่
   สำคัญไม่แพ้เรื่อง technical เลย

---

## แบบฝึกหัดท้ายบท

1. เขียนไฟล์ `vcpkg.json` (Manifest Mode) ที่ประกาศ dependency 3 ตัว: `fmt`,
   `nlohmann-json`, และ `catch2` (เตรียมไว้ใช้ใน Part 93) โดยระบุเวอร์ชันขั้นต่ำของแต่ละตัว
2. เขียนไฟล์ `conanfile.py` ที่ทำหน้าที่เดียวกันกับข้อ 1 (ประกาศ dependency 3 ตัวเดียวกัน) โดย
   ใช้ `CMakeDeps`/`CMakeToolchain` เป็น generator
3. ทดลองรัน `conan profile detect --force` บนเครื่องของตัวเอง แล้วอธิบายว่าค่า
   `compiler.libcxx` ที่ตรวจพบคืออะไร เชื่อมโยงกับความรู้เรื่อง ABI Compatibility จาก Part 39
4. เขียน `CMakeLists.txt` ที่ใช้ `find_package(Threads REQUIRED)` (system-level) ควบคู่กับ
   `find_package(fmt REQUIRED)` (สมมติว่ามาจาก vcpkg/Conan) ในโปรเจกต์เดียว ตามแนวทาง
   "Workflow แบบผสม" ในหัวข้อ 92.6
5. อธิบายด้วยคำพูดตัวเอง (เขียนเป็นย่อหน้าสั้นๆ) ว่าทำไมบริษัทที่ทำงานด้าน Defense/Finance
   ที่มักมีนโยบาย Air-gapped Network ถึงมักตั้ง Private Registry ของ vcpkg/Conan ขึ้นมาเอง
   ภายในองค์กร แทนที่จะพึ่ง ConanCenter/vcpkg ports สาธารณะโดยตรง
6. (โบนัส) ทดลองติดตั้ง vcpkg หรือ Conan บนเครื่องของตัวเอง (ที่มี internet เชื่อมต่อปกติ ไม่ใช่
   sandbox แบบบทเรียนนี้) แล้วลองติดตั้ง library `fmt` จริงจนสำเร็จ เขียนโปรแกรมเล็กๆ ที่ใช้
   `fmt::format` แล้ว build ผ่าน CMake ให้ผ่านจริง บันทึกผลลัพธ์ที่ได้เทียบกับที่บทเรียนนี้ทำ
   ไม่สำเร็จเพราะข้อจำกัดของ sandbox

### แนวทางเฉลยข้อ 1

```json
{
  "name": "module-h-demo",
  "version": "1.0.0",
  "dependencies": [
    { "name": "fmt", "version>=": "11.0.0" },
    { "name": "nlohmann-json", "version>=": "3.11.0" },
    { "name": "catch2", "version>=": "3.4.0" }
  ]
}
```

จุดสำคัญที่ต้องสังเกต: ชื่อ package ใน vcpkg ใช้ `nlohmann-json` (มีขีดกลาง) ไม่ใช่
`nlohmann_json` (ขีดล่าง) แบบที่ Conan ใช้ — นี่คือตัวอย่างเล็กๆ ที่แสดงให้เห็นว่าแม้แนวคิดจะ
คล้ายกัน แต่รายละเอียดของแต่ละระบบนิเวศ (ecosystem) ก็ยังต่างกันในระดับ syntax ต้องอ่าน
documentation ของแต่ละตัวให้ละเอียดก่อนใช้งานจริงเสมอ ห้ามเดาชื่อ package เอง

### แนวทางเฉลยข้อ 2

```python
from conan import ConanFile
from conan.tools.cmake import CMakeToolchain, CMakeDeps, cmake_layout

class ModuleHDemoRecipe(ConanFile):
    settings = "os", "compiler", "build_type", "arch"
    generators = "CMakeDeps", "CMakeToolchain"

    def requirements(self):
        self.requires("fmt/11.0.2")
        self.requires("nlohmann_json/3.11.3")
        self.requires("catch2/3.5.4")

    def layout(self):
        cmake_layout(self)
```

โน้ตสำคัญ: `conanfile.py` ให้ความยืดหยุ่นที่ `conanfile.txt` ทำไม่ได้ เช่น ถ้าต้องการ requires
library ต่างกันตามเงื่อนไข (เช่น เฉพาะบน Windows ถึงจะต้องการ library ตัวหนึ่ง) สามารถเขียน
logic ภาษา Python ปกติภายในเมธอด `requirements()` ได้เลย ซึ่งเป็นข้อได้เปรียบสำคัญของ Conan
เหนือ vcpkg ในกรณีที่ dependency ของโปรเจกต์ซับซ้อนขึ้นตามแพลตฟอร์มหรือ configuration

### แนวทางเฉลยข้อ 4

```cmake
cmake_minimum_required(VERSION 3.20)
project(MixedDepsApp LANGUAGES CXX)

# pthread มากับ OS อยู่แล้ว (Part 31) -- ใช้ Find Module มาตรฐานของ CMake ตรงๆ
find_package(Threads REQUIRED)

# fmt สมมติว่ามาจาก vcpkg/Conan ผ่าน toolchain file ตอน configure
# (cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=.../vcpkg.cmake)
find_package(fmt REQUIRED)

add_executable(app main.cpp)
target_link_libraries(app PRIVATE Threads::Threads fmt::fmt)
```

`main.cpp` ตัวอย่างที่ใช้ทั้งสอง dependency ร่วมกัน:

```cpp
#include <fmt/core.h>
#include <thread>
#include <cstdio>

int main() {
    std::thread t([] {
        fmt::print("Hello from a std::thread using fmt!\n");
    });
    t.join();
    return 0;
}
```

ประเด็นสำคัญของเฉลยนี้คือการยืนยันแนวคิดจากหัวข้อ 92.6 ด้วยโค้ดจริง: `find_package` ทั้ง
สองบรรทัดมีรูปแบบเดียวกันทุกประการ แม้ `Threads` จะมาจาก CMake's built-in Find Module ที่
ค้นหาใน system โดยตรง ส่วน `fmt` มาจาก config file ที่ Package Manager ภายนอกสร้างให้ —
ผู้เขียนโค้ดในไฟล์นี้ไม่จำเป็นต้องรู้หรือสนใจความต่างนั้นเลย

### แนวทางเฉลยข้อ 5

องค์กรในสาย Defense/Finance มักถูกกำหนดด้วยนโยบายด้าน Security ที่เข้มงวดว่าเครื่องที่ใช้
พัฒนา/build ซอฟต์แวร์ (โดยเฉพาะ Build Server ใน CI/CD) **ห้ามเชื่อมต่อ internet สาธารณะ
โดยตรง** (เรียกว่า Air-gapped Network) เหตุผลหลักมีอย่างน้อย 3 ข้อ:

1. **ป้องกัน Supply Chain Attack**: ถ้า Build Server ดึง dependency จาก ConanCenter/vcpkg
   ports สาธารณะโดยตรงทุกครั้งที่ build และมีใครแฮ็ก package บนนั้นสำเร็จ (เคยเกิดเหตุการณ์
   จริงลักษณะนี้กับ Package Registry ของหลายภาษามาแล้ว) โค้ดอันตรายจะถูกดึงเข้ามาปนใน
   Production Build โดยตรงทันที การตัดขาด internet และควบคุมทุก dependency ผ่าน registry
   ภายในที่ผ่านการตรวจสอบ (Audit) ก่อนเท่านั้น ลดความเสี่ยงนี้ได้มาก
2. **ควบคุมและตรวจสอบทุก dependency ได้ (Compliance)**: อุตสาหกรรม Finance/Defense
   มักต้องผ่านการตรวจสอบตามมาตรฐาน (เช่น ต้องพิสูจน์ได้ว่าโค้ดทุกบรรทัดที่ใช้ในระบบผ่านการ
   Audit ด้าน Security แล้ว) การมี Private Registry ที่เก็บเฉพาะเวอร์ชันของ library ที่ผ่านการ
   ตรวจสอบและอนุมัติแล้วเท่านั้น ทำให้ตอบคำถามผู้ตรวจสอบ (Auditor) ได้ง่ายกว่าการอนุญาตให้
   ทุกเครื่องดึงอะไรก็ได้จาก internet สาธารณะโดยตรง
3. **ความเสถียรและ Availability**: ถ้า Build Server ทุกเครื่องพึ่งพา ConanCenter/vcpkg ports
   สาธารณะโดยตรง แล้ว service เหล่านั้น down หรือช้าชั่วคราว (เกิดขึ้นได้เสมอกับ service
   สาธารณะที่มีคนใช้งานทั่วโลก) การ build ทั้งองค์กรจะหยุดชะงักไปด้วย การมี Binary Cache/
   Private Registry ภายในองค์กรทำให้ build เร็วและเสถียรกว่ามาก เพราะดึงจากเครือข่ายภายใน
   ที่ควบคุมคุณภาพเองได้

ปัญหาที่เราเจอจริงในบทเรียนนี้ (`403 Forbidden` จาก proxy ตอนพยายามเข้าถึง ConanCenter
และ GitHub) เป็นตัวอย่างเล็กๆ ของสถานการณ์แบบเดียวกันนี้พอดี — sandbox ที่ใช้รันบทเรียนถูก
จำกัด network ด้วยเหตุผลด้าน Security เช่นเดียวกับที่องค์กรใหญ่ๆ ทำกับ Build Server ของตัวเอง

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจว่าทำไม C++ ถึงต้องการ Package Manager เฉพาะทาง (vcpkg/Conan) ต่างจากภาษาที่มี
  npm/pip เพราะ C++ compile เป็น native code ที่ผูกกับ compiler/ABI ของแต่ละแพลตฟอร์ม
  อย่างแนบแน่น
- ติดตั้งและทดลองใช้งาน **vcpkg** จริง (bootstrap สำเร็จ, ติดตั้ง helper port สำเร็จ) และเข้าใจ
  แนวคิด Port-based กับ Triplet
- ติดตั้งและทดลองใช้งาน **Conan** จริง (ตรวจจับ Profile ของเครื่องสำเร็จ) และเข้าใจแนวคิด
  Profile-based Dependency Resolution
- เห็นด้วยตาตัวเองว่าความพยายามดาวน์โหลด library จริงทั้งสองเครื่องมือล้มเหลวด้วยเหตุผล
  เดียวกัน (network ถูกจำกัดใน sandbox) และเข้าใจว่านี่คือปัญหาจริงที่ CI/CD แบบ Air-gapped
  เจอบ่อยในโลกการทำงานจริง ไม่ใช่แค่ทฤษฎี
- เปรียบเทียบ vcpkg กับ Conan อย่างละเอียด และเปรียบเทียบทั้งคู่กับแนวทาง System Package
  ที่ใช้มาตลอดหลักสูตร ผ่านการทดลองจริงกับ `find_package(ZLIB)` ที่ compile และรันสำเร็จ
- เข้าใจว่า `find_package` + `target_link_libraries` เป็น interface เดียวกันไม่ว่า library จะ
  มาจากแหล่งใด ทำให้เปลี่ยนแหล่งที่มาของ dependency ได้โดยแทบไม่กระทบโค้ด CMake เดิม

ตอนนี้เรามีเครื่องมือครบสำหรับจัดการโครงสร้างโปรเจกต์ (CMake) และ dependency ภายนอก
(vcpkg/Conan) แล้ว ใน **Part 93** เราจะนำทักษะทั้งหมดนี้มาใช้ติดตั้งและเชื่อมต่อ **Testing
Framework** สองตัวที่สำคัญที่สุดในวงการ C++ คือ **Google Test** และ **Catch2** เพื่อเขียน
Unit Test ที่ป้องกัน Regression และทำให้ Refactor โค้ดได้อย่างมั่นใจ

**ต่อไป:** [Part 93 — Unit Testing ด้วย Google Test และ Catch2](./part-093-unit-testing.md)
