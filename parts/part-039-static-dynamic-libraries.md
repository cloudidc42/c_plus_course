# Part 39: Static และ Dynamic Library (Step 305–312)

> Module C — Systems Programming ด้วย C บน Linux | Part 39 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 305–312
> Part ก่อนหน้า: [Part 38 — Valgrind และ Memory Leak Detection](./part-038-valgrind.md) | Part ถัดไป: [Part 40 — โปรเจกต์ Mini Key-Value Store Engine](./part-040-kv-store-project.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายความแตกต่างพื้นฐานระหว่าง **Static Library (.a)** และ **Shared/Dynamic Library (.so)**
   ทั้งในแง่ของเวลาที่เกิดการ link และผลลัพธ์ที่ได้
2. สร้าง Static Library ด้วยคำสั่ง `ar` และ link โปรแกรมเข้ากับมันได้
3. สร้าง Shared Library ด้วย `gcc -shared -fPIC` พร้อมเข้าใจว่าทำไมต้องมี `-fPIC`
4. ตั้งชื่อไฟล์ `.so` ตามธรรมเนียมมาตรฐาน (`soname`, `real name`, `linker name`) และเข้าใจว่า
   symlink 3 ชั้นนี้มีไว้ทำอะไร
5. วินิจฉัยและแก้ปัญหา **"cannot open shared object file"** ด้วย `LD_LIBRARY_PATH`, `rpath`,
   และ `ldconfig`
6. ใช้ `ldd`, `nm`, `objdump`, `readelf` ตรวจสอบ dependency และ symbol ของไฟล์ library/executable
7. อธิบายแนวคิดพื้นฐานของ **Symbol Versioning** และเหตุผลที่ไลบรารีระดับระบบอย่าง glibc ต้องใช้
8. ตัดสินใจเลือกใช้ Static หรือ Dynamic Library ให้เหมาะกับสถานการณ์จริง โดยชั่งน้ำหนักระหว่าง
   ขนาดไฟล์ ความสะดวกในการอัปเดต และ performance

---

## 39.1 ทำไมต้องมี Library และความแตกต่างระหว่างสองแบบ (Step 305)

ย้อนกลับไป Part 17–18 เราเรียนรู้การแยกโปรแกรมเป็นหลายไฟล์ (`math_utils.h/.c` + `main.c`)
แล้วคอมไพล์แยกกันเป็น `.o` ก่อนค่อย link รวมเป็น executable ปัญหาคือถ้าโค้ด `math_utils` ตัวนี้
มีประโยชน์และอยากใช้ซ้ำใน**หลายโปรเจกต์** เราจะต้อง copy ไฟล์ `.c`/`.h` ไปวางในทุกโปรเจกต์
หรือ compile `math_utils.o` ใหม่ทุกครั้งที่เริ่มโปรเจกต์ใหม่ — **Library** คือคำตอบของปัญหานี้:
การแพ็ก object code ที่ compile เสร็จแล้วให้เป็นไฟล์ก้อนเดียวที่หลายโปรเจกต์เรียกใช้ร่วมกันได้
โดยไม่ต้องมี source code หรือ compile ใหม่อีก

Linux มี Library อยู่ 2 แบบหลัก ซึ่งต่างกันที่ **"เวลาที่โค้ดของ library ถูกรวมเข้ากับโปรแกรม"**:

```
Static Library (.a)                    Shared/Dynamic Library (.so)
─────────────────────                  ──────────────────────────
math_utils.c → math_utils.o            math_utils.c → math_utils.o (-fPIC)
      │                                       │
      ▼                                       ▼
  libmathutils.a                        libmathutils.so
      │                                       │
      ▼ (ตอน LINK)                            ▼ (ตอน RUNTIME)
┌─────────────────┐                    ┌─────────────────┐
│    program      │  โค้ดของ           │    program      │  แค่ "จด" ว่า
│  ┌───────────┐  │  math_utils ถูก    │                 │  ต้องพึ่ง .so
│  │ main.o    │  │  copy เข้ามา       │  ┌───────────┐  │  ตัวไหน แล้วให้
│  │ math_utils│  │  รวมในไฟล์         │  │ main.o    │  │  OS โหลด .so
│  │  (copy)   │  │  executable       │  └───────────┘  │  มา "เสียบ" ให้
│  └───────────┘  │  เดียวจบ           │                 │  ตอนรันจริง
└─────────────────┘                    └─────────────────┘
                                               │
                                        ต้องมี libmathutils.so
                                        อยู่ในเครื่องตอนรันด้วย!
```

| ประเด็น | Static Library (`.a`) | Shared Library (`.so`) |
|---|---|---|
| เวลาที่โค้ดถูกรวมเข้าโปรแกรม | **ตอน link** (compile-time) | **ตอนรัน** (runtime, โดย dynamic linker) |
| โค้ดของ library อยู่ในไฟล์ executable หรือไม่ | อยู่ — executable มีโค้ดครบในตัวเอง | ไม่อยู่ — executable แค่จดชื่อไว้ ต้องหา `.so` ตอนรัน |
| ขนาดไฟล์ executable | ใหญ่กว่า (มีโค้ด library รวมอยู่) | เล็กกว่ามาก |
| การอัปเดต library | ต้อง **link โปรแกรมใหม่ทั้งหมด** เมื่อ library เปลี่ยน | แค่เปลี่ยนไฟล์ `.so` โปรแกรมเดิมใช้ตัวใหม่ได้ทันที (ไม่ต้อง recompile) |
| การแชร์ RAM เมื่อรันหลายโปรเซส | แต่ละโปรเซสมีโค้ด library แยกชุดของตัวเอง | โปรเซสหลายตัวที่ใช้ `.so` เดียวกัน **แชร์หน่วยความจำโค้ดส่วนนี้ร่วมกันได้** (OS โหลดแค่ครั้งเดียว) |
| ความเร็วตอนเริ่มโปรแกรม (startup) | เร็วกว่าเล็กน้อย (ไม่ต้องหา/โหลด `.so` เพิ่ม) | ช้ากว่าเล็กน้อย (ต้องให้ dynamic linker หา `.so` และ resolve symbol ก่อนรัน) |
| Deployment | copy executable ไฟล์เดียวไปที่ไหนก็รันได้เลย | ต้องมั่นใจว่าเครื่องปลายทางมี `.so` ที่ต้องการอยู่ด้วย |
| นามสกุลไฟล์บน Linux | `.a` (archive) | `.so` (shared object) |

> เทียบเคียงกับ Windows: `.a` ↔ `.lib` (static), `.so` ↔ `.dll` (dynamic) — แนวคิดเหมือนกันทุก
> ระบบปฏิบัติการ เพียงแต่ชื่อไฟล์และเครื่องมือต่างกัน

ตลอด Part นี้จะใช้โปรเจกต์ `math_utils` จาก Part 17–18 (ไฟล์ `math_utils.h`, `math_utils.c`,
`main.c` ชุดเดิมทุกประการ) เป็นตัวอย่างในการแพ็กเป็นทั้ง static และ dynamic library

---

## 39.2 สร้าง Static Library ด้วย `ar` (Step 306)

**`ar`** (archiver) เป็นเครื่องมือมาตรฐานของ Unix ที่มีมาตั้งแต่ยุคแรกๆ (เก่าแก่กว่า `make` ด้วยซ้ำ)
หน้าที่ของมันคือ **รวมไฟล์ `.o` หลายไฟล์เข้าเป็นไฟล์ archive เดียว** — Static Library ก็คือ
ไฟล์ archive แบบนี้นั่นเอง

```bash
# ขั้นที่ 1: compile เป็น .o ตามปกติ (เหมือน Part 17-18 ทุกประการ)
gcc -Wall -Wextra -Wpedantic -std=c17 -c math_utils.c -o math_utils.o

# ขั้นที่ 2: รวม .o เข้าเป็น static library ด้วย ar
ar rcs libmathutils.a math_utils.o
```

flag ของ `ar` ที่ใช้:

| Flag | ความหมาย |
|---|---|
| `r` | insert/replace — เพิ่มไฟล์เข้า archive (หรือแทนที่ถ้ามีชื่อซ้ำอยู่แล้ว) |
| `c` | create — สร้าง archive ใหม่โดยไม่ต้องเตือนถ้ายังไม่มีไฟล์นี้อยู่ก่อน |
| `s` | เขียน **index** ของ symbol ทั้งหมดในไฟล์ archive ไว้ (ทำให้ linker ค้นหาฟังก์ชันเจอเร็วขึ้น
        มาก โดยไม่ต้องไล่เปิดทุก `.o` ในไฟล์ทีละตัว — เทียบเท่ากับการรัน `ranlib` แยกต่างหาก) |

### ธรรมเนียมการตั้งชื่อไฟล์: ทำไมต้องขึ้นต้นด้วย `lib`

สังเกตว่าตั้งชื่อว่า **`libmathutils.a`** ไม่ใช่ `mathutils.a` เฉยๆ — นี่คือ**ธรรมเนียมบังคับ**ของ
Linux/Unix ที่ linker คาดหวัง เมื่อสั่ง link ด้วย flag `-lmathutils` (จะอธิบายในหัวขัดถัดไป)
linker จะไปค้นหาไฟล์ชื่อ `libmathutils.a` หรือ `libmathutils.so` โดยอัตโนมัติ (เติม `lib` นำหน้า
และเติมนามสกุลให้เอง) — ถ้าตั้งชื่อไฟล์ผิดธรรมเนียมนี้ จะ link ด้วย `-l` ไม่ได้เลย ต้องระบุ path
เต็มแทน

### ตรวจสอบเนื้อหาใน archive

```bash
ar t libmathutils.a
```

```
math_utils.o
```

`ar t` (table of contents) แสดงรายชื่อไฟล์ `.o` ทั้งหมดที่อยู่ใน archive — ในตัวอย่างนี้มีแค่ไฟล์
เดียว แต่ archive จริงมักรวม `.o` หลายสิบหรือหลายร้อยไฟล์เข้าด้วยกัน (เช่น `libc.a` ที่รวม
implementation ของทุกฟังก์ชันมาตรฐานของภาษา C ไว้ในไฟล์เดียว)

```bash
ar tv libmathutils.a
```

```
rw-r--r-- 0/0   2832 Jan  1 00:00 1970 math_utils.o
```

`ar tv` (verbose) แสดงรายละเอียดเพิ่มเติม (permission, ขนาดไฟล์) ของแต่ละ member ใน archive

### Link โปรแกรมเข้ากับ Static Library

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 main.c -L. -lmathutils -o prog_static
./prog_static
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

- **`-L.`**: บอก linker ให้ค้นหา library เพิ่มเติมในโฟลเดอร์ปัจจุบัน (`.`) — ถ้าไม่ใส่ linker
  จะค้นหาแค่ในโฟลเดอร์ระบบมาตรฐานเท่านั้น (เช่น `/usr/lib`, `/usr/local/lib`) และหา
  `libmathutils.a` ที่เราเพิ่งสร้างไม่เจอ
- **`-lmathutils`**: บอกให้ link กับ library ชื่อ `mathutils` (linker จะเติม `lib` นำหน้าและ
  ต่อท้ายด้วย `.a` หรือ `.so` ให้อัตโนมัติตามที่อธิบายไว้ข้างบน)

ผลลัพธ์ `prog_static` คือ executable ที่ **มีโค้ดของ `math_utils` ทั้งหมดฝังอยู่ข้างในไฟล์แล้ว**
ตรวจสอบด้วย `file`:

```bash
file prog_static
```

```
prog_static: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, BuildID[...], for GNU/Linux 3.2.0, not stripped
```

> **สังเกต**: ถึงจะบอกว่า "dynamically linked" ก็ตาม แต่นั่นหมายถึงโปรแกรมยังต้องพึ่ง `libc.so`
> (ไลบรารีมาตรฐานของระบบ) อยู่ดี ซึ่งเป็นเรื่องปกติมาก — คำว่า static/dynamic ในบทเรียนนี้พูดถึง
> เฉพาะไลบรารีของเราเอง (`libmathutils`) เท่านั้น โค้ดของ `math_utils` ถูกฝังเข้าไปใน
> `prog_static` เรียบร้อยแล้วไม่ว่ากรณีใด (จะพิสูจน์ให้เห็นชัดในหัวข้อ 39.4)

---

## 39.3 สร้าง Shared Library ด้วย `gcc -shared -fPIC` (Step 307)

การสร้าง Shared Library มีขั้นตอนที่ต่างออกไปเล็กน้อยแต่สำคัญมาก:

```bash
# ขั้นที่ 1: compile ด้วย -fPIC (สำคัญมาก อธิบายด้านล่าง)
gcc -Wall -Wextra -Wpedantic -std=c17 -fPIC -c math_utils.c -o math_utils_pic.o

# ขั้นที่ 2: สร้าง .so ด้วย gcc -shared
gcc -shared -Wl,-soname,libmathutils.so.1 -o libmathutils.so.1.0.0 math_utils_pic.o
```

### `-fPIC` คืออะไร และทำไมจำเป็น

**`-fPIC`** ย่อจาก **Position Independent Code** สั่งให้ compiler สร้าง machine code ที่**ไม่
ผูกติดกับตำแหน่งหน่วยความจำที่แน่นอน** เหตุผลที่ต้องการคุณสมบัตินี้คือ Shared Library ตัวเดียวกัน
อาจถูกโหลดไปอยู่ที่ **ตำแหน่งหน่วยความจำต่างกัน** ในแต่ละโปรเซสที่ใช้มัน (โดยเฉพาะเมื่อระบบเปิด
ใช้ ASLR — Address Space Layout Randomization เพื่อความปลอดภัย) โค้ดที่ compile แบบปกติ (ไม่ใส่
`-fPIC`) มักอ้างอิงตำแหน่งตัวแปร/ฟังก์ชันด้วย **absolute address** ที่ตายตัว ซึ่งใช้ไม่ได้ถ้าโค้ด
ถูกโหลดไปอยู่คนละที่ในแต่ละครั้ง ส่วนโค้ดที่ compile ด้วย `-fPIC` จะอ้างอิงด้วย **relative
address** (ระยะห่างจากตำแหน่งปัจจุบัน) แทน ทำให้ทำงานถูกต้องไม่ว่าจะถูกโหลดไปอยู่ที่ไหนก็ตาม

> **กฎทองของ Part นี้**: object file ที่จะเอาไปทำ Shared Library **ต้อง** compile ด้วย `-fPIC`
> เสมอ ส่วน object file สำหรับ Static Library **ไม่จำเป็นต้องมี** `-fPIC` (แต่ใส่ก็ไม่ผิดอะไร
> เพียงแค่ทำให้โค้ดช้าลงเล็กน้อยโดยไม่มีประโยชน์ เพราะ static library ไม่ได้ต้องการคุณสมบัตินี้)

### ทำความเข้าใจ `-Wl,-soname,libmathutils.so.1`

`-Wl,` เป็นวิธีส่ง flag ผ่าน `gcc` ตรงไปให้ **linker** (`ld`) โดยตรง (`gcc` เป็นแค่ตัวห่อหุ้ม
`ld` อีกที) ส่วน `soname` คือชื่อ **"identity"** ของ shared library ตัวนี้ที่จะถูกฝังไว้ข้างใน
ตัวไฟล์ `.so` เอง — โปรแกรมที่ link กับ library นี้จะจดจำ **soname** นี้ไว้ (ไม่ใช่ชื่อไฟล์จริงที่
ใช้ตอน compile) ซึ่งเป็นกลไกสำคัญที่ทำให้อัปเดต library เป็นเวอร์ชันย่อยได้โดยไม่ต้อง link
โปรแกรมใหม่ (จะอธิบายเพิ่มในหัวข้อ 39.7)

### ธรรมเนียมการตั้งชื่อ 3 ชั้นของ Shared Library

Shared Library บน Linux นิยมมีชื่อไฟล์ 3 แบบที่เชื่อมกันด้วย symlink:

```bash
ln -sf libmathutils.so.1.0.0 libmathutils.so.1   # symlink: soname -> real name
ln -sf libmathutils.so.1     libmathutils.so     # symlink: linker name -> soname
```

```bash
ls -la libmathutils*
```

```
-rw-r--r-- 1 root root  3110 ...  libmathutils.a
-rwxr-xr-x 1 root root 15480 ...  libmathutils.so.1.0.0
lrwxrwxrwx 1 root root    21 ...  libmathutils.so.1 -> libmathutils.so.1.0.0
lrwxrwxrwx 1 root root    17 ...  libmathutils.so   -> libmathutils.so.1
```

| ชื่อ | เรียกว่า | ใครใช้ | ตัวอย่าง |
|---|---|---|---|
| `libmathutils.so.1.0.0` | **Real Name** | ไฟล์จริงที่มีเนื้อโค้ดอยู่ (Major.Minor.Patch) | ไฟล์นี้เท่านั้นที่มีขนาดจริง |
| `libmathutils.so.1` | **SONAME** | โปรแกรมที่ link เสร็จแล้วจะจำชื่อนี้ไว้ใน metadata ของตัวเอง (ระบบปฏิบัติการใช้ตอนรัน) | ฝังอยู่ใน field `NEEDED` ของ executable |
| `libmathutils.so` | **Linker Name** | ใช้เฉพาะตอน **compile/link** เท่านั้น (`-lmathutils` จะมาเจอชื่อนี้) โปรแกรมที่ compile เสร็จแล้วไม่ยุ่งกับชื่อนี้อีกเลย | นักพัฒนาใช้ตอน `gcc ... -lmathutils` |

การแยกชื่อ 3 ชั้นนี้คือกลไกสำคัญที่ทำให้ทีมพัฒนาสามารถ**อัปเดต patch version** ของ library
(เช่น `1.0.0` → `1.0.1` เพื่อแก้บั๊กเล็กน้อยโดยไม่เปลี่ยน API) แล้วแค่เปลี่ยน symlink
`libmathutils.so.1` ให้ชี้ไปยังไฟล์เวอร์ชันใหม่ **โปรแกรมเก่าที่ link ไว้แล้วใช้งานต่อได้ทันที
โดยไม่ต้อง recompile หรือ relink เลย** เพราะโปรแกรมจดจำแค่ `libmathutils.so.1` (SONAME) ไว้
ไม่ได้จดจำเลขเวอร์ชันแบบเต็ม

---

## 39.4 Link โปรแกรมกับทั้งสองแบบ และพิสูจน์ความแตกต่าง (Step 308)

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 main.c -L. -lmathutils -o prog_dynamic
./prog_dynamic
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

**สังเกตจุดสำคัญ**: คำสั่ง compile นี้**เหมือนกับตอนสร้าง `prog_static` ทุกตัวอักษร**
(`-L. -lmathutils`)! เมื่อทั้ง `libmathutils.a` และ `libmathutils.so` อยู่ในโฟลเดอร์เดียวกัน
`gcc`/`ld` จะ**เลือก `.so` เป็นค่า default เสมอ** (ถือว่า dynamic linking เป็นตัวเลือกที่ดีกว่า
โดยทั่วไป) ถ้าต้องการบังคับให้ link แบบ static ทั้งที่มี `.so` อยู่ด้วย ต้องสั่งชัดเจน:

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 main.c -Wl,-Bstatic -lmathutils -Wl,-Bdynamic -L. -o prog_force_static
```

`-Wl,-Bstatic` และ `-Wl,-Bdynamic` เป็น "สวิตช์" ที่ส่งผลกับ `-l` ที่ตามมาหลังจากนั้นเท่านั้น
(ในตัวอย่างนี้บังคับให้ `-lmathutils` ใช้แบบ static แต่ `libc` ยังคง dynamic ตามปกติ)

### พิสูจน์ด้วยขนาดไฟล์และ `ldd`

```bash
ls -la libmathutils.a libmathutils.so.1.0.0 prog_static prog_dynamic
```

```
-rw-r--r-- 1 root root  3110 ...  libmathutils.a
-rwxr-xr-x 1 root root 15480 ...  libmathutils.so.1.0.0
-rwxr-xr-x 1 root root 16256 ...  prog_dynamic
-rwxr-xr-x 1 root root 16448 ...  prog_static
```

ในตัวอย่างเล็กๆ นี้ความต่างของขนาดยังไม่มาก (`prog_static` ใหญ่กว่า `prog_dynamic` แค่ 192
bytes) เพราะ `math_utils` มีโค้ดน้อยมาก แต่ในสถานการณ์จริงที่ library มีขนาดหลายเมกะไบต์ (เช่น
`libssl`, `libopencv`) ความต่างของขนาด executable ระหว่าง static กับ dynamic link จะเห็นชัดเจน
มาก (dynamic จะเล็กกว่าหลายสิบหรือหลายร้อยเท่า เพราะไม่ต้องแบกโค้ด library มาด้วย)

`ldd` (List Dynamic Dependencies) คือเครื่องมือสำคัญที่สุดในการตรวจสอบว่า executable ต้องพึ่ง
`.so` ตัวไหนบ้าง:

```bash
ldd ./prog_dynamic
```

```
	linux-vdso.so.1 (0x00007f48b81a5000)
	libmathutils.so.1 => not found
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f48b7e00000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f48b81a7000)
```

สังเกตบรรทัด **`libmathutils.so.1 => not found`** — นี่คือปัญหาสำคัญที่จะแก้ในหัวข้อถัดไป

```bash
ldd ./prog_static
```

```
	linux-vdso.so.1 (0x00007f...)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f...)
	/lib64/ld-linux-x86-64.so.2 (0x00007f...)
```

`prog_static` **ไม่มี `libmathutils` ปรากฏใน `ldd` เลย** เพราะโค้ดทั้งหมดถูกฝังเข้าไปในไฟล์
executable ตั้งแต่ตอน link แล้ว — ไม่ต้องพึ่งไฟล์ภายนอกใดๆ เพิ่มเติมนอกจาก `libc` ที่เป็นไลบรารี
มาตรฐานของระบบ (ซึ่งแทบทุกโปรแกรม C ต้องพึ่งอยู่ดีไม่ว่าจะ static หรือ dynamic link library
ของตัวเองก็ตาม — glibc นั้นแทบไม่มีใครใช้แบบ static เพราะมีข้อจำกัดหลายอย่าง)

---

## 39.5 แก้ปัญหา "cannot open shared object file" (Step 309)

ลองรัน `prog_dynamic` ตรงๆ:

```bash
./prog_dynamic
```

```
./prog_dynamic: error while loading shared libraries: libmathutils.so.1: cannot open shared object file: No such file or directory
```

นี่คือ error ที่โปรแกรมเมอร์ C ทุกคนต้องเจอแน่นอนสักวันหนึ่ง สาเหตุคือ **dynamic linker
(`ld.so`) ไม่รู้ว่าจะหา `libmathutils.so.1` ได้จากที่ไหน** เพราะมันไม่ได้อยู่ในโฟลเดอร์มาตรฐานของ
ระบบ (`/lib`, `/usr/lib`) และไม่มีการบอกที่อยู่เพิ่มเติมไว้เลย ต่างจาก **compile time** ที่เรา
บอก linker ผ่าน `-L.` ได้ แต่ **runtime** เป็นคนละขั้นตอนกันโดยสิ้นเชิงและต้องบอกแยกต่างหาก

### วิธีที่ 1: `LD_LIBRARY_PATH` (เหมาะกับการทดสอบชั่วคราว)

```bash
LD_LIBRARY_PATH=. ./prog_dynamic
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

`LD_LIBRARY_PATH` เป็น environment variable ที่บอก dynamic linker ให้ค้นหา `.so` เพิ่มเติมใน
โฟลเดอร์ที่ระบุ (คั่นหลายโฟลเดอร์ด้วย `:` เหมือน `PATH`) — ใช้งานง่ายและเหมาะกับการทดสอบระหว่าง
พัฒนา แต่**ไม่แนะนำสำหรับ production** เพราะเป็นการตั้งค่าระดับ shell session ที่ผู้ใช้ปลายทาง
ต้องจำไปตั้งเองทุกครั้ง ลืมตั้งเมื่อไรโปรแกรมก็รันไม่ได้ทันที

```bash
LD_LIBRARY_PATH=. ldd ./prog_dynamic
```

```
	linux-vdso.so.1 (0x00007f79a1bc9000)
	libmathutils.so.1 => ./libmathutils.so.1 (0x00007f79a1bb7000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f79a1800000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f...)
	/lib64/ld-linux-x86-64.so.2 (0x00007f79a1bcb000)
```

เมื่อตั้ง `LD_LIBRARY_PATH` แล้ว `ldd` เจอ `libmathutils.so.1` ที่ `./libmathutils.so.1` ทันที

### วิธีที่ 2: `rpath` (ฝังที่อยู่ไว้ในตัว executable เอง — เหมาะกับ production มากกว่า)

```bash
gcc -Wall -Wextra -Wpedantic -std=c17 main.c -L. -lmathutils -Wl,-rpath,'$ORIGIN' -o prog_rpath
./prog_rpath
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

รันได้ทันทีโดย**ไม่ต้องตั้ง `LD_LIBRARY_PATH` เลย**! `-Wl,-rpath,PATH` ฝัง path ที่จะค้นหา `.so`
ไว้ **ข้างในตัว executable เอง** ตอน compile เลย ตัวแปรพิเศษ **`$ORIGIN`** หมายถึง "โฟลเดอร์ที่
ไฟล์ executable ตัวนี้ตั้งอยู่จริงตอนรัน" (ไม่ใช่ตอน compile) ทำให้ย้ายทั้งโฟลเดอร์ (executable +
`.so`) ไปที่ไหนก็ยังหากันเจอเสมอ เพราะระยะห่างสัมพัทธ์ยังเท่าเดิม — เทคนิคนี้เป็นที่นิยมมากในการ
แจกจ่ายซอฟต์แวร์ที่รวม library เฉพาะของตัวเองไปด้วย (bundled library)

> **ข้อควรระวัง**: ต้องใส่ `$ORIGIN` เป็น**เครื่องหมายคำพูดเดี่ยว** (`'$ORIGIN'`) ในคำสั่ง shell
> เสมอ ไม่เช่นนั้น bash จะพยายามตีความ `$ORIGIN` เป็นตัวแปร shell (ซึ่งไม่มีค่า) ก่อนที่จะส่งต่อ
> ให้ `gcc` ทำให้ rpath ที่ฝังเข้าไปว่างเปล่าโดยไม่มี error ใดๆ เตือนเลย

### วิธีที่ 3: `ldconfig` (เหมาะกับ library ที่ติดตั้งระบบทั้งเครื่อง)

สำหรับ library ที่ต้องการติดตั้งให้**ทุกโปรแกรมในเครื่องใช้ร่วมกันได้** (ไม่ผูกกับโปรเจกต์ใด
โปรเจกต์หนึ่ง) วิธีมาตรฐานคือติดตั้งเข้าโฟลเดอร์ระบบแล้วรัน `ldconfig`:

```bash
sudo cp libmathutils.so.1.0.0 /usr/local/lib/
sudo ldconfig    # อัปเดต cache ของ dynamic linker (/etc/ld.so.cache)
```

`ldconfig` จะสแกนโฟลเดอร์มาตรฐานทั้งหมด (รวมถึงที่ตั้งค่าไว้ใน `/etc/ld.so.conf`) แล้วสร้าง
symlink SONAME → real name ให้อัตโนมัติ พร้อมอัปเดต cache ที่ dynamic linker ใช้ค้นหาไฟล์อย่าง
รวดเร็ว (แทนที่จะต้องสแกนทุกโฟลเดอร์ทุกครั้งที่รันโปรแกรม) หลังรัน `ldconfig` แล้วโปรแกรมใดๆ
ในเครื่องก็รัน `./prog_dynamic` ได้ทันทีโดยไม่ต้องตั้งอะไรเพิ่มเลย

### สรุปเปรียบเทียบ 3 วิธี

| วิธี | ขอบเขตผลกระทบ | เหมาะกับ |
|---|---|---|
| `LD_LIBRARY_PATH` | เฉพาะ shell session ที่ตั้งค่า | ทดสอบระหว่างพัฒนา |
| `-Wl,-rpath` | ฝังถาวรในตัว executable | แจกจ่ายโปรแกรมพร้อม library ของตัวเอง |
| `ldconfig` + ติดตั้งระบบ | ทั้งเครื่อง ทุกโปรแกรม | library ระบบที่ใช้ร่วมกันหลายโปรเจกต์ |

---

## 39.6 ตรวจสอบ Dependency และ Symbol: `ldd`, `nm`, `objdump` (Step 310)

นอกจาก `ldd` ที่เห็นไปแล้ว ยังมีเครื่องมืออีก 2 ตัวที่ใช้บ่อยมากในการตรวจสอบไฟล์ library/object:

### `nm` — แสดงรายชื่อ Symbol ทั้งหมด

```bash
nm math_utils.o
```

```
000000000000001d t mu_abs_int
0000000000000034 T mu_add
0000000000000175 T mu_average
0000000000000000 B mu_call_count
00000000000000bf T mu_gcd
...
```

ตัวอักษรกลาง (`T`, `t`, `B`) บอก**ประเภท**ของ symbol:

| ตัวอักษร | ความหมาย |
|---|---|
| `T` (ตัวใหญ่) | ฟังก์ชัน public ใน text section (โค้ด) — เรียกจากไฟล์อื่นได้ (external linkage) |
| `t` (ตัวเล็ก) | ฟังก์ชัน `static` ใน text section — เรียกจากไฟล์อื่นไม่ได้ (internal linkage, ทบทวนจาก Part 17) เช่น `mu_abs_int` |
| `B`/`b` | ตัวแปร global ใน BSS section (ตัวแปรที่ยังไม่ได้กำหนดค่าเริ่มต้น หรือกำหนดเป็น 0) เช่น `mu_call_count` |
| `U` | Undefined — symbol ที่ไฟล์นี้**เรียกใช้** แต่ไม่ได้ implement เอง (ต้องมาจากไฟล์อื่นตอน link) |

สังเกตว่า `mu_abs_int` (ฟังก์ชัน `static` จาก Part 17) ปรากฏเป็นตัวพิมพ์เล็ก `t` ยืนยันว่ามันถูก
ซ่อนจากภายนอกจริงๆ ตามที่ตั้งใจไว้

สำหรับ Shared Library ใช้ `nm -D` (Dynamic symbols) เพื่อดูเฉพาะ symbol ที่**เปิดเผยให้ไฟล์อื่น
เรียกใช้ได้ตอน runtime**:

```bash
nm -D libmathutils.so.1.0.0
```

```
                 w _ITM_deregisterTMCloneTable
                 w __cxa_finalize
                 w __gmon_start__
0000000000001133 T mu_add
0000000000001274 T mu_average
0000000000004010 B mu_call_count
00000000000011be T mu_gcd
0000000000001330 T mu_get_call_count
0000000000001212 T mu_is_prime
0000000000001173 T mu_power
0000000000001317 T mu_reset_call_count
0000000000001154 T mu_subtract
```

ทุกฟังก์ชัน public ของ `math_utils` ปรากฏครบ (`w` คือ "weak symbol" ของระบบที่ไม่เกี่ยวกับโค้ด
ของเรา สามารถมองข้ามได้) — นี่คือวิธีตรวจสอบว่า Shared Library ที่สร้างขึ้น **export ฟังก์ชันที่
ต้องการครบถ้วนหรือไม่** โดยไม่ต้องเขียนโปรแกรมทดสอบแยกต่างหาก

### `objdump` — วิเคราะห์ไฟล์ ELF แบบละเอียด

```bash
objdump -T libmathutils.so.1.0.0
```

ให้ผลลัพธ์คล้าย `nm -D` แต่มีรายละเอียดเพิ่มเติม (เช่น เวอร์ชันของ symbol ซึ่งจะพูดถึงในหัวข้อ
ถัดไป) `objdump` ยังใช้ดู disassembly (แปลง machine code กลับเป็น assembly) ได้ด้วย แนวคิด
เดียวกับที่เคยดูผ่าน `gcc -S` ใน Part 1:

```bash
objdump -d libmathutils.so.1.0.0 | grep -A 5 "<mu_add>:"
```

```
0000000000001133 <mu_add>:
    1133:	f3 0f 1e fa          	endbr64
    1137:	55                   	push   %rbp
    1138:	48 89 e5             	mov    %rsp,%rbp
    113b:	89 7d ec             	mov    %edi,-0x14(%rbp)
    113e:	89 75 e8             	mov    %esi,-0x18(%rbp)
```

---

## 39.7 Symbol Versioning เบื้องต้น (Step 311)

ปัญหาที่ SONAME (หัวข้อ 39.3) ยังแก้ไม่ได้ทั้งหมดคือ: ถ้า library อัปเดตแล้ว**ลบฟังก์ชันบางตัวออก
หรือเปลี่ยน signature** โปรแกรมเก่าที่ยัง link กับ SONAME เดิมจะพังทันที **Symbol Versioning**
คือกลไกที่ช่วยให้ library หนึ่งไฟล์**มีหลายเวอร์ชันของฟังก์ชันเดียวกันอยู่พร้อมกันได้** เพื่อรองรับ
ทั้งโปรแกรมเก่าและโปรแกรมใหม่พร้อมกัน

### ตัวอย่างจริงจาก glibc

glibc (`libc.so.6`) เป็นตัวอย่างที่ดีที่สุดของการใช้ symbol versioning จริงจัง:

```bash
objdump -T /lib/x86_64-linux-gnu/libc.so.6 | grep -m 6 "GLIBC_2\.[0-9]* .*printf"
```

```
0000000000138410 g    DF .text	0000000000000031  GLIBC_2.4   __vswprintf_chk
0000000000138330 g    DF .text	000000000000001a  GLIBC_2.4   __vfwprintf_chk
00000000001382d0 g    DF .text	000000000000001a  GLIBC_2.8   __vasprintf_chk
```

คอลัมน์ `GLIBC_2.4`, `GLIBC_2.8` คือ**เวอร์ชันของ symbol** — บอกว่าฟังก์ชันนี้ถูกเพิ่ม/เปลี่ยน
แปลงพฤติกรรมตั้งแต่ glibc เวอร์ชันไหน โปรแกรมที่ compile ด้วย glibc เก่าจะถูกผูก (bind) เข้ากับ
symbol เวอร์ชันเก่าโดยอัตโนมัติ ในขณะที่โปรแกรมที่ compile ใหม่กว่าจะได้ symbol เวอร์ชันใหม่ —
**ทั้งสองแบบทำงานถูกต้องพร้อมกันได้บน libc.so.6 ไฟล์เดียวกัน** นี่คือเหตุผลสำคัญที่ทำให้โปรแกรม
ที่ compile ไว้นานแล้วยังรันได้ปกติบน Linux เวอร์ชันใหม่ๆ โดยไม่ต้อง recompile

### สร้าง Symbol Versioning ในไลบรารีของเราเอง

ทำผ่านไฟล์ **Version Script** ที่ส่งให้ linker ตอนสร้าง `.so`:

```
# mathutils.map
MATHUTILS_1.0 {
    global:
        mu_add;
        mu_subtract;
        mu_power;
        mu_gcd;
        mu_is_prime;
        mu_average;
        mu_reset_call_count;
        mu_get_call_count;
        mu_call_count;
    local:
        *;
};
```

`global:` คือรายชื่อ symbol ที่ต้องการ **export ออกไปให้ไฟล์อื่นเรียกใช้ได้** ภายใต้ป้ายชื่อ
เวอร์ชัน `MATHUTILS_1.0` ส่วน `local: *;` คือ**ปิดกั้น symbol อื่นที่ไม่ได้ระบุไว้ทั้งหมด** ไม่ให้
เห็นจากภายนอก (เป็นการบังคับใช้ Encapsulation ระดับ library ทบทวนแนวคิดจาก Part 17 ที่ใช้
`static` ทำแบบเดียวกันในระดับไฟล์)

```bash
gcc -shared -fPIC -Wl,--version-script=mathutils.map \
    -Wl,-soname,libmathutils.so.1 -o libmathutils_v.so.1.0.0 math_utils_pic.o

readelf --version-info libmathutils_v.so.1.0.0
```

ผลลัพธ์จริงที่ได้:

```
Version symbols section '.gnu.version' contains 15 entries:
 Addr: 0x000000000000058c  Offset: 0x0000058c  Link: 4 (.dynsym)
  000:   0 (*local*)       1 (*global*)      1 (*global*)      1 (*global*)
  004:   1 (*global*)      2 (MATHUTILS_1.0)   2 (MATHUTILS_1.0)   2 (MATHUTILS_1.0)
  008:   2 (MATHUTILS_1.0)   2 (MATHUTILS_1.0)   2 (MATHUTILS_1.0)   2 (MATHUTILS_1.0)
  00c:   2 (MATHUTILS_1.0)   2 (MATHUTILS_1.0)   2 (MATHUTILS_1.0)

Version definition section '.gnu.version_d' contains 2 entries:
 Addr: 0x00000000000005b0  Offset: 0x000005b0  Link: 5 (.dynstr)
  000000: Rev: 1  Flags: BASE  Index: 1  Cnt: 1  Name: libmathutils.so.1
  0x001c: Rev: 1  Flags: none  Index: 2  Cnt: 1  Name: MATHUTILS_1.0
```

ยืนยันว่าทุกฟังก์ชันของเราถูกผูกเข้ากับป้ายชื่อเวอร์ชัน `MATHUTILS_1.0` เรียบร้อยแล้ว ถ้าในอนาคต
ต้องแก้ signature ของ `mu_add` แบบไม่ backward-compatible ก็สามารถเพิ่ม block ใหม่
`MATHUTILS_2.0 { ... } MATHUTILS_1.0;` ที่มีฟังก์ชัน `mu_add` เวอร์ชันใหม่ควบคู่ไปกับเวอร์ชันเก่า
ในไฟล์ `.so` เดียวกันได้ — เทคนิคนี้ใช้จริงในไลบรารีระดับ production แทบทุกตัวที่ต้องรักษาความ
เข้ากันได้ย้อนหลัง (backward compatibility) ในระยะยาว แต่สำหรับโปรเจกต์ขนาดเล็กถึงกลางส่วนใหญ่
การใช้แค่ SONAME (หัวข้อ 39.3) ก็เพียงพอแล้วในทางปฏิบัติ

---

## 39.8 เปรียบเทียบข้อดี-ข้อเสีย และสรุป (Step 312)

| ประเด็น | Static Library | Dynamic/Shared Library |
|---|---|---|
| **ขนาดไฟล์ executable** | ใหญ่กว่า (โค้ดถูกฝังไว้) | เล็กกว่ามาก |
| **การอัปเดตโค้ด library** | ต้อง recompile/relink โปรแกรมทั้งหมดที่ใช้ | แค่แทนที่ไฟล์ `.so` โปรแกรมเดิมใช้ต่อได้ทันที |
| **Deployment** | ง่ายที่สุด — copy ไฟล์เดียวจบ ไม่ต้องพึ่งอะไรเพิ่ม | ต้องดูแลให้เครื่องปลายทางมี `.so` ที่ตรงกันด้วย |
| **การแชร์ RAM ระหว่างหลายโปรเซส** | ไม่มี — แต่ละโปรเซสมีสำเนาโค้ดของตัวเอง | มี — OS โหลดโค้ดที่ใช้ร่วมกันแค่ครั้งเดียว |
| **ความเร็วตอน startup** | เร็วกว่าเล็กน้อย | ช้ากว่าเล็กน้อย (ต้อง resolve symbol ตอนรัน) |
| **Performance ตอนรัน (function call)** | เร็วกว่าเล็กน้อยมาก (เรียกตรงๆ ไม่ผ่านชั้น indirection) | มี overhead เล็กน้อยจาก PLT/GOT (Procedure Linkage Table) แต่ในทางปฏิบัติแทบไม่รู้สึกต่างเลยสำหรับโปรแกรมทั่วไป |
| **Security patch** | ต้องรอ vendor แต่ละรายอัปเดต binary ของตัวเอง | อัปเดต `.so` ตัวเดียว ทุกโปรแกรมในเครื่องได้ patch ทันที (นี่คือเหตุผลที่ช่องโหว่ร้ายแรงระดับโลกอย่าง Heartbleed ถูกแก้ไขได้เร็วในระบบที่ใช้ shared library) |
| **เหมาะกับ** | เครื่องมือ command-line ที่ต้องการ portable สูง, embedded system ที่ไม่มี dynamic linker, ต้องการควบคุมทุกอย่างในไฟล์เดียว | โปรแกรมขนาดใหญ่ที่แชร์ library ร่วมกันหลายตัว, ระบบที่ต้องอัปเดตบ่อย, ไลบรารีระบบปฏิบัติการ |

### แนวทางตัดสินใจในทางปฏิบัติ

- ถ้ากำลังสร้าง **เครื่องมือ command-line เดี่ยวๆ** ที่แจกจ่ายให้ผู้ใช้ทั่วไป (เช่น CLI tool)
  และไม่อยากให้ผู้ใช้ต้องกังวลเรื่อง dependency เลย — **static link** มักเป็นตัวเลือกที่ปลอดภัย
  และง่ายกว่า (โปรแกรมภาษา Go นิยม static link แทบทุกอย่างด้วยเหตุผลนี้)
- ถ้ากำลังพัฒนา **library ที่จะถูกใช้โดยหลายโปรแกรมในระบบเดียวกัน** (เช่น `libssl`, `libcurl`)
  หรือเป็นส่วนหนึ่งของ **package ที่ต้องอัปเดต security patch บ่อย** — **shared library**
  คือตัวเลือกที่เหมาะสมกว่ามาก
- ระบบปฏิบัติการ Linux ส่วนใหญ่ (รวมถึง distro อย่าง Ubuntu, Debian) **บังคับให้ใช้ dynamic
  linking กับ glibc เป็นค่า default** เพราะประโยชน์ด้าน security patch และการแชร์ RAM สำคัญกว่า
  ข้อเสียเรื่อง startup time เล็กน้อยมาก
- ในโลกจริง ทีมพัฒนาจำนวนมากเลือก**ผสมทั้งสองแบบ**: static link กับ library เฉพาะทางที่ไม่ค่อย
  เปลี่ยนแปลงและอยากควบคุมเวอร์ชันเอง (vendoring) แต่ dynamic link กับ library ระบบพื้นฐาน
  (`libc`, `libm`, `libpthread`) ที่ควรใช้เวอร์ชันของระบบปฏิบัติการเสมอ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืม `-fPIC` ตอนสร้าง object file สำหรับ Shared Library** — จะได้ error ทำนอง
   `relocation R_X86_64_PC32 against symbol ... can not be used when making a shared object`
   จาก linker ทันที เพราะโค้ดที่ไม่ใช่ Position Independent ใช้ทำ `.so` ไม่ได้
2. **ลืม `-L.` เมื่อ library อยู่ในโฟลเดอร์ปัจจุบัน** — จะได้ error
   `/usr/bin/ld: cannot find -lmathutils` เพราะ linker ค้นหาแค่โฟลเดอร์ระบบมาตรฐานเป็นค่า default
   เท่านั้น
3. **ลืมว่า compile-time linking (`-L`) กับ run-time linking (`LD_LIBRARY_PATH`) เป็นคนละเรื่อง
   กัน** — บาง `.so` compile ผ่านได้ปกติด้วย `-L.` แต่พอรันโปรแกรมจริงกลับเจอ
   `cannot open shared object file` เพราะไม่มีใครบอก dynamic linker ตอนรันเลยว่าจะหา `.so`
   จากที่ไหน (ต้องแก้ด้วยวิธีในหัวข้อ 39.5)
4. **ใช้ `$ORIGIN` ใน rpath โดยไม่ครอบด้วย single quote** — bash ตีความ `$ORIGIN` เป็นตัวแปร
   environment (ที่ไม่มีค่า) ก่อนส่งให้ `gcc` ทำให้ rpath ที่ฝังเข้าไปว่างเปล่าโดยไม่มี error
   เตือน ต้องเขียนเป็น `-Wl,-rpath,'$ORIGIN'` เสมอ
5. **แก้ไข library แล้วลืมว่าโปรแกรมที่ static link ไว้แล้วจะไม่เห็นการเปลี่ยนแปลงเลย** — ถ้าลืม
   ลักษณะนี้ของ static library อาจ debug อยู่นานว่า "ทำไมแก้โค้ดแล้วพฤติกรรมไม่เปลี่ยน" ทั้งที่
   ต้องแค่ relink โปรแกรมใหม่ (ไม่จำเป็นต้อง recompile ทั้ง `main.c` ถ้าไม่ได้แก้ `main.c` เอง
   แค่ relink object ใหม่กับ library เวอร์ชันใหม่ก็พอ)
6. **ตั้งชื่อไฟล์ library ผิดธรรมเนียม** (เช่น `mathutils.a` แทนที่จะเป็น `libmathutils.a`) แล้ว
   สงสัยว่าทำไม `-lmathutils` หา library ไม่เจอ ทั้งที่ไฟล์อยู่ตรงหน้าชัดๆ — ต้องขึ้นต้นด้วย `lib`
   เสมอเพื่อให้ `-l` flag ทำงานได้

---

## แบบฝึกหัดท้ายบท

1. สร้าง static library ชื่อ `libstringutils.a` จากโปรเจกต์ `string_utils` ใน Part 17-18
   (แบบฝึกหัดข้อ 1 ของ Part 18) แล้ว link โปรแกรม `main.c` เข้ากับมันด้วย `-L. -lstringutils`
2. สร้าง shared library `libstringutils.so` จากไฟล์ชุดเดียวกัน (ต้อง compile ด้วย `-fPIC` ใหม่)
   ตั้งชื่อ 3 ชั้นให้ถูกต้องตามธรรมเนียม (real name, soname, linker name) แล้ว link โปรแกรมเข้า
   กับมัน จากนั้นพิสูจน์ด้วย `ldd` ว่า error "cannot open shared object file" เกิดขึ้นจริงเมื่อ
   รันตรงๆ โดยไม่ตั้งค่าอะไรเพิ่ม แล้วแก้ด้วยเทคนิค `-Wl,-rpath,'$ORIGIN'`
3. ทดลองแก้ไขฟังก์ชันใน `string_utils.c` (เช่นเปลี่ยนพฤติกรรมของ `reverse()`) แล้ว **build
   shared library ใหม่แต่ไม่ recompile โปรแกรม `main.c`** (แค่ relink `.so` ทับของเดิม) รัน
   โปรแกรมเดิมอีกครั้งแล้วสังเกตว่าพฤติกรรมเปลี่ยนไปตามโค้ดใหม่ทันที จากนั้นลองทำแบบเดียวกันกับ
   static library เทียบดูว่าพฤติกรรมต่างกันอย่างไร (คำใบ้: static ต้อง relink `main.o` ใหม่ด้วย
   เสมอ)
4. ใช้ `nm -D` ตรวจสอบว่า `libstringutils.so` export ฟังก์ชัน `static` ที่เขียนไว้ภายในไฟล์
   (ถ้ามี) ออกไปให้เห็นจากภายนอกหรือไม่ อธิบายว่าทำไมถึงเป็นเช่นนั้น
5. เขียน Version Script (`stringutils.map`) สำหรับ `libstringutils.so` ที่ export เฉพาะฟังก์ชัน
   public เท่านั้น (ปิดกั้นฟังก์ชันอื่นทั้งหมดด้วย `local: *;`) แล้วตรวจสอบด้วย `readelf
   --version-info` ว่า version script ทำงานถูกต้อง
6. เปรียบเทียบขนาดไฟล์ระหว่าง executable ที่ link แบบ static เต็มรูปแบบ (`gcc -static`) กับแบบ
   dynamic ปกติ ของโปรแกรม `main.c` + `math_utils` จาก Part นี้ อธิบายว่าทำไมส่วนต่างถึงมากกว่า
   ตอนที่ link แค่ `libmathutils` แบบ static/dynamic เฉยๆ (คำใบ้: `gcc -static` บังคับให้ `libc`
   ทั้งก้อนถูกฝังเข้ามาด้วย ไม่ใช่แค่ `libmathutils`)

### แนวทางเฉลยข้อ 1

สมมติใช้ไฟล์ `string_utils.h`/`string_utils.c` และ `main.c` ชุดเดียวกับแบบฝึกหัดข้อ 1 ของ
Part 18 (มีฟังก์ชัน `reverse()` และ `is_palindrome()`):

```bash
# 1) compile เป็น .o ตามปกติ (ไม่ต้องมี -fPIC เพราะเป็น static library)
gcc -Wall -Wextra -Wpedantic -std=c17 -c string_utils.c -o string_utils.o

# 2) รวมเป็น static library
ar rcs libstringutils.a string_utils.o

# 3) ตรวจสอบเนื้อหา
ar t libstringutils.a

# 4) link โปรแกรมเข้ากับ static library
gcc -Wall -Wextra -Wpedantic -std=c17 main.c -L. -lstringutils -o prog_static
./prog_static
```

ผลลัพธ์:

```
string_utils.o

reverse("Hello") = "olleH"
is_palindrome("level") = 1
is_palindrome("Racecar") = 1
is_palindrome("Hello") = 0
is_palindrome("A") = 1
```

ยืนยันด้วย `ldd` ว่าไม่มี `libstringutils` ปรากฏเลย (ต่างจากกรณี dynamic library ในข้อ 2):

```bash
$ ldd ./prog_static
	linux-vdso.so.1 (0x...)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x...)
	/lib64/ld-linux-x86-64.so.2 (0x...)
```

โค้ดของ `string_utils` ทั้งหมดถูกฝังเข้าไปใน `prog_static` เรียบร้อยแล้วตั้งแต่ตอน link จึงไม่ต้อง
พึ่งไฟล์ `.a`/`.so` ภายนอกใดๆ อีกเลยตอนรัน (`libstringutils.a` เองก็สามารถลบทิ้งได้หลังจาก link
เสร็จแล้ว โปรแกรมยังรันได้ปกติ — ต่างจาก shared library ที่ต้องเก็บ `.so` ไว้ตลอดไป)

### แนวทางเฉลยข้อ 2

```bash
# 1) compile ด้วย -fPIC
gcc -Wall -Wextra -Wpedantic -std=c17 -fPIC -c string_utils.c -o string_utils_pic.o

# 2) สร้าง .so พร้อม soname
gcc -shared -Wl,-soname,libstringutils.so.1 \
    -o libstringutils.so.1.0.0 string_utils_pic.o

# 3) ตั้งชื่อ 3 ชั้นตามธรรมเนียม
ln -sf libstringutils.so.1.0.0 libstringutils.so.1
ln -sf libstringutils.so.1     libstringutils.so

# 4) link โปรแกรมเข้ากับ .so
gcc -Wall -Wextra -Wpedantic -std=c17 main.c -L. -lstringutils -o prog
```

พิสูจน์ปัญหา:

```bash
$ ./prog
./prog: error while loading shared libraries: libstringutils.so.1: cannot open shared object file: No such file or directory

$ ldd ./prog
	linux-vdso.so.1 (0x...)
	libstringutils.so.1 => not found
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x...)
	/lib64/ld-linux-x86-64.so.2 (0x...)
```

แก้ด้วย rpath:

```bash
$ gcc -Wall -Wextra -Wpedantic -std=c17 main.c -L. -lstringutils -Wl,-rpath,'$ORIGIN' -o prog_fixed
$ ./prog_fixed
reverse("Hello") = "olleH"
is_palindrome("level") = 1
...

$ ldd ./prog_fixed
	linux-vdso.so.1 (0x...)
	libstringutils.so.1 => ./libstringutils.so.1 (0x...)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x...)
	/lib64/ld-linux-x86-64.so.2 (0x...)
```

`prog_fixed` รันได้ทันทีโดยไม่ต้องตั้ง `LD_LIBRARY_PATH` เลย เพราะ path ค้นหาถูกฝังไว้ในตัว
executable เองแล้วผ่าน `$ORIGIN` (โฟลเดอร์เดียวกับที่ไฟล์ `prog_fixed` ตั้งอยู่จริงตอนรัน) — จุด
สำคัญของเฉลยนี้คือการแยกให้ชัดว่า **error "cannot open shared object file" เป็นปัญหาระดับ
runtime ไม่ใช่ compile-time** ถึงแม้คำสั่ง `gcc` ตอน build จะผ่านสมบูรณ์แบบไม่มี error ใดๆ เลย
ก็ตาม

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจความแตกต่างพื้นฐานระหว่าง Static Library (`.a`, รวมโค้ดตอน link) และ Shared Library
  (`.so`, รวมโค้ดตอนรัน) พร้อมตารางเปรียบเทียบทุกมิติ
- สร้าง Static Library ด้วย `ar rcs` และ Shared Library ด้วย `gcc -shared -fPIC` จากโปรเจกต์
  `math_utils` เดียวกัน พร้อมเข้าใจความหมายของ `-fPIC` และธรรมเนียมชื่อ 3 ชั้นของ `.so`
  (real name / SONAME / linker name)
- วินิจฉัยและแก้ปัญหา "cannot open shared object file" ได้ 3 วิธี: `LD_LIBRARY_PATH`, rpath
  (`$ORIGIN`), และการติดตั้งระบบด้วย `ldconfig`
- ใช้ `ldd`, `nm`, `objdump`, `readelf` ตรวจสอบ dependency และ symbol ของไฟล์ library/executable
  ได้อย่างมั่นใจ
- เข้าใจแนวคิด Symbol Versioning ผ่านตัวอย่างจริงจาก glibc และสร้าง version script ของตัวเอง
- ตัดสินใจเลือก static หรือ dynamic library ให้เหมาะกับสถานการณ์จริงได้ โดยพิจารณาขนาดไฟล์
  การอัปเดต และ performance

ตลอด Module C (Part 26-39) เราได้เรียนรู้ Systems Programming บน Linux อย่างครบวงจร ตั้งแต่
Process, Signal, IPC, Thread, Socket, mmap, ไปจนถึง Debugging และ Library Management ใน
**Part 40** ซึ่งเป็น Part สุดท้ายของ Module C เราจะนำความรู้ทั้งหมดนี้มาประกอบร่างเป็นโปรเจกต์
ใหญ่ชิ้นแรกของหลักสูตร: **Mini Key-Value Store Engine** ที่รองรับหลาย client พร้อมกันผ่าน TCP
Socket เหมือน Redis ย่อส่วน

**ต่อไป:** [Part 40 — โปรเจกต์ Mini Key-Value Store Engine](./part-040-kv-store-project.md)
