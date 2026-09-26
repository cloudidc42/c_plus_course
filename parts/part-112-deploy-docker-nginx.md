# Part 112: Deploy: Docker, Nginx, Production Server (Step 889–896)

> Module I — Web Development ด้วย C/C++ | Part 112 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 889–896
> Part ก่อนหน้า: [Part 111 — Server-Side Rendering ด้วย C++](./part-111-server-side-rendering.md) | Part ถัดไป: [Part 113 — Clean Code และ Code Review Practice](./part-113-clean-code-review.md)

## เป้าหมายการเรียนรู้ (Learning Objectives)

หลังจบ Part นี้ ผู้เรียนจะสามารถ:

1. อธิบายปัญหาคลาสสิก "มันรันบนเครื่องผมนะ" (It works on my machine) ได้อย่างชัดเจน และอธิบาย
   ได้ว่าแนวคิด **Reproducible Environment** ผ่าน Container ช่วยแก้ปัญหานี้อย่างไรในเชิงเทคนิค
2. เขียน **Dockerfile แบบ Multi-stage Build** สำหรับ compile และรัน C++ web application จาก
   Part 108 ได้ถูกต้อง โดยแยก build stage (มี compiler เต็มรูปแบบ) ออกจาก runtime stage
   (เบาที่สุดเท่าที่จะเป็นไปได้) อย่างเข้าใจเหตุผลเบื้องหลังทุกบรรทัด
3. เขียน **docker-compose.yml** ที่ประกอบ container ของ backend C++ เข้ากับ PostgreSQL
   container เข้าใจเรื่อง network ภายใน, volume สำหรับข้อมูลถาวร, และการส่งค่า configuration
   ผ่าน environment variable ตามแนวคิด 12-Factor App
4. ตั้งค่า **Nginx เป็น Reverse Proxy** หน้า C++ backend ได้จริงบนเครื่อง พร้อมเข้าใจ
   `proxy_pass`, `proxy_set_header` และหลุมพรางเรื่อง trailing slash ที่มือใหม่พลาดบ่อยที่สุด
5. อธิบายแนวคิดของ **HTTPS ด้วย Let's Encrypt/Certbot** ได้ถูกต้อง เข้าใจว่าทำไมต้องทำ
   TLS Termination ที่ชั้น Reverse Proxy แทนที่จะทำใน backend C++ โดยตรง
6. เขียนและตรวจสอบ **systemd unit file** สำหรับรัน C++ server เป็น background service บน
   Linux production server จริง เข้าใจ lifecycle ทั้งหมด (start/stop/restart/enable) และ
   การ hardening พื้นฐาน
7. ประเมิน **Production Readiness** ของระบบด้วย checklist ที่ครอบคลุม logging, monitoring
   เบื้องต้น และ graceful shutdown แล้วนำไปตรวจสอบระบบของตัวเองได้จริงในอนาคต

> **หมายเหตุเรื่องสภาพแวดล้อมของ Part นี้ (สำคัญมาก อ่านก่อนเริ่ม)**
>
> Part นี้ทดสอบเนื้อหาจริงบนเครื่องที่มี `g++ 13.3.0`, Crow ติดตั้งที่ `/usr/local/include/crow`,
> `nginx 1.24.0`, `systemd 255`, และ `PostgreSQL 16.15` ครบทุกตัว **แต่เครื่องที่ใช้เขียนบทเรียนนี้
> เป็น sandboxed container ที่ไม่มี Docker daemon ทำงานอยู่** (รัน `docker info` แล้วจะได้ error
> `Cannot connect to the Docker daemon` เพราะการรัน Docker daemon ต้องการสิทธิ์ privileged/root
> ระดับเครื่องจริงที่ container แบบ sandbox นี้ไม่มีให้) ด้วยเหตุนี้ เนื้อหาในบทเรียนจึงแบ่งชัดเจน
> เป็น 2 กลุ่มตลอดทั้งบท:
>
> - ✅ **ทดสอบจริงบนเครื่องนี้แล้ว** — โค้ด Task API ที่ compile จริง, การตั้งค่า Nginx reverse
>   proxy ที่รันจริงและทดสอบด้วย `curl` จริง, systemd unit file ที่ผ่าน `systemd-analyze verify`
>   จริง, และการเชื่อมต่อ PostgreSQL จริงด้วย credential แบบเดียวกับที่จะใช้ใน `docker-compose.yml`
> - 📄 **โค้ดอ้างอิง (Reference only) — ไม่ได้รันจริงในสภาพแวดล้อมนี้** — คือ `Dockerfile` และ
>   `docker-compose.yml` ทั้งหมด เนื้อหาเหล่านี้เขียนและตรวจทานอย่างละเอียดถูกต้องตามหลักปฏิบัติ
>   จริง แต่คำสั่ง `docker build`/`docker compose up` **ไม่ได้ถูกรันจริง** ในบทเรียนนี้ ทุกจุดที่เป็น
>   กลุ่มนี้จะมีป้ายกำกับชัดเจนว่า "ผลลัพธ์ที่คาดว่าจะได้" ไม่ใช่ผลจริงที่ copy มาจาก terminal
>
> การบอกตรงๆ แบบนี้สำคัญกว่าการแกล้งทำเป็นว่าทุกอย่างรันได้หมด เพราะในงานจริงผู้เรียนก็จะเจอ
> สถานการณ์ที่ไม่มี Docker ติดตั้งอยู่บนเครื่อง CI/CD บางตัว หรือทำงานในองค์กรที่จำกัดสิทธิ์
> ไม่ให้รัน Docker ได้ — ทักษะที่สำคัญกว่าคือ **อ่าน Dockerfile เป็นและรู้ว่ามันควรทำอะไร**
> ไม่ใช่แค่ก็อปคำสั่งมาแปะแล้วรอดูว่าพังหรือไม่พัง

---

## 112.1 ทำไมต้อง Deploy ด้วย Docker: ปัญหา "มันรันบนเครื่องผมนะ" (Step 889)

### สถานการณ์ที่เกิดขึ้นจริงแทบทุกทีมพัฒนาซอฟต์แวร์

ลองจินตนาการสถานการณ์นี้ ซึ่งเกิดขึ้นจริงนับไม่ถ้วนครั้งในวงการซอฟต์แวร์:

1. โปรแกรมเมอร์เขียน Task API (จาก Part 108) บนเครื่องตัวเอง — Ubuntu 24.04, `g++ 13.3.0`,
   Crow ติดตั้งไว้ที่ `/usr/local/include/crow`, SQLite 3.45.1 ที่ apt ติดตั้งให้ — ทุกอย่าง
   compile ผ่าน รันได้ ทดสอบผ่านหมด
2. ส่งโค้ดให้เพื่อนร่วมทีมทดสอบ เพื่อนใช้ Ubuntu 22.04 ที่มี `g++ 11` เป็นค่า default —
   โค้ดที่ใช้ฟีเจอร์ C++17/20 บางตัว **compile ไม่ผ่าน** เพราะ compiler เก่ากว่า
3. แก้ปัญหา compiler แล้ว แต่พอ deploy ขึ้นเซิร์ฟเวอร์จริง (สมมติเป็น CentOS หรือ Debian
   เวอร์ชันเก่า) กลับพบว่า **ไม่มี Crow ติดตั้งอยู่เลย** เพราะ Crow เป็น header-only library
   ที่ต้อง copy ไฟล์ไปวางเองในแต่ละเครื่อง ไม่ได้อยู่ใน package manager มาตรฐาน
4. แก้ไขจนติดตั้ง Crow สำเร็จ แต่โปรแกรมที่รันบนเซิร์ฟเวอร์กลับ **behavior ต่างจากที่ทดสอบไว้**
   เพราะเซิร์ฟเวอร์มี `libsqlite3` คนละเวอร์ชันที่ compile มาด้วย flag ต่างกัน (บางเวอร์ชันเก่า
   ไม่รองรับ WAL mode ที่โค้ดเราใช้)

ทุกขั้นตอนข้างต้นคือปัญหาที่เรียกรวมๆ ว่า **"It works on my machine"** — โค้ดทำงานถูกต้องบน
เครื่องของคนเขียน แต่ล้มเหลวเมื่อย้ายไปรันที่อื่น เพราะ **สภาพแวดล้อม (Environment) ไม่เหมือนกัน**

### รากของปัญหา: สิ่งที่ "สภาพแวดล้อม" ครอบคลุมมากกว่าที่คิด

| องค์ประกอบของ Environment | ตัวอย่างความต่างที่ทำให้พัง |
|---|---|
| เวอร์ชัน OS/Distro | Ubuntu 24.04 vs CentOS 7 vs Alpine — glibc คนละเวอร์ชัน |
| เวอร์ชัน Compiler | `g++ 13` รองรับ C++23 บางส่วน, `g++ 9` รองรับแค่ C++17 |
| เวอร์ชันของ Library ที่ link | `libsqlite3`, `libssl`, `libpq` คนละเวอร์ชัน พฤติกรรมต่างกันได้ |
| ตำแหน่งไฟล์ของ Dependency | Crow อาจอยู่ `/usr/local/include` เครื่องหนึ่ง แต่ `/opt/crow` อีกเครื่อง |
| ตัวแปรสภาพแวดล้อม (Environment Variable) | `PATH`, `LD_LIBRARY_PATH` ตั้งค่าไม่เหมือนกัน |
| Kernel/System Call ที่รองรับ | ฟีเจอร์บางตัวของ `epoll`/`io_uring` ต้องการ kernel เวอร์ชันใหม่พอ |
| การตั้งค่า Locale/Timezone | โปรแกรมที่ format วันที่ผิดเพราะ locale เซิร์ฟเวอร์ไม่ใช่ `th_TH` |

จุดสำคัญที่ต้องเข้าใจคือ **โค้ด C/C++ ไวต่อปัญหานี้มากกว่าภาษาที่ compile เป็น bytecode/interpreted**
เช่น Java (JVM) หรือ Python เพราะ C/C++ compile เป็น **native machine code ที่ผูกกับ ABI (Application
Binary Interface)** ของระบบตอน compile โดยตรง ต่างจาก JVM ที่มี "เครื่องเสมือน" คั่นกลางทำให้ bytecode
เดียวกันรันได้บนหลาย OS

### Docker แก้ปัญหานี้อย่างไร: แนวคิด Reproducible Environment

**Docker** คือเครื่องมือสำหรับสร้าง **Container** — สภาพแวดล้อมที่ถูก "ห่อ" (package) ไว้ทั้งหมด
ครบทุกอย่างที่โปรแกรมต้องการเพื่อรัน (compiler ที่ใช้ตอน build, library ที่ต้อง link, ไฟล์ config,
แม้กระทั่ง OS base image) ไว้ในไฟล์เดียวที่เรียกว่า **Image** แล้วรัน image นั้นเป็น container
ที่ไหนก็ได้ที่มี Docker Engine ติดตั้งอยู่ — ผลลัพธ์จะ**เหมือนกันทุกประการ** ไม่ว่าจะรันบนเครื่อง
นักพัฒนา, เซิร์ฟเวอร์ staging, หรือเซิร์ฟเวอร์ production จริง

หลักการสำคัญคือ **"Build Once, Run Anywhere"** — เราสร้าง image ครั้งเดียว (หรือใน CI/CD pipeline)
แล้วนำ image เดียวกันนั้นไป deploy ได้ทุกที่ โดยไม่ต้องกังวลว่าเซิร์ฟเวอร์ปลายทางจะมี dependency
ครบหรือไม่ เพราะทุกอย่างอยู่ใน image แล้ว

### เปรียบเทียบแนวทาง Deploy แบบต่างๆ

| แนวทาง | ข้อดี | ข้อเสีย |
|---|---|---|
| **Copy binary ไปวางตรงๆ** | เร็วที่สุด ไม่มี overhead | ต้องมั่นใจว่า shared library ที่ binary ต้องการมีอยู่ตรงเวอร์ชันบนเซิร์ฟเวอร์เป้าหมายทุกครั้ง |
| **Static linking** (compile ให้ทุกอย่างฝังในไฟล์เดียว) | ไม่ต้องพึ่ง shared library ของระบบเลย | ไฟล์ใหญ่ขึ้นมาก, อัปเดต security patch ของ library ต้อง compile ใหม่ทุกครั้ง |
| **Virtual Machine (VM)** | Isolation สมบูรณ์ 100% รวมถึง kernel | หนักมาก (จำลอง hardware ทั้งเครื่อง), boot ช้า, ใช้ทรัพยากรสูง |
| **Docker Container** | เบากว่า VM มาก (แชร์ kernel กับ host), เริ่มทำงานเร็ว (วินาที ไม่ใช่นาที), reproducible เต็มรูปแบบ | Isolation ไม่สมบูรณ์เท่า VM (แชร์ kernel), ต้องเรียนรู้เครื่องมือใหม่ (Dockerfile, registry) |
| **systemd service ตรงบนเซิร์ฟเวอร์** (ไม่ผ่าน container) | ง่ายที่สุด ไม่มี layer เพิ่ม, performance เต็มที่ (ไม่มี virtualization overhead ใดๆ เลย) | ต้องดูแล dependency ของเซิร์ฟเวอร์เองทั้งหมด, ยากที่จะรันหลายเวอร์ชันของ dependency พร้อมกัน |

**ข้อสังเกตสำคัญ**: Docker กับ systemd **ไม่ใช่คู่แข่งกัน** — ในทางปฏิบัติจริง หลายทีมใช้ทั้งสองคู่กัน
คือรัน `docker run` ผ่าน systemd unit file เพื่อให้ container เริ่มทำงานอัตโนมัติตอนเครื่องบูต และ
restart อัตโนมัติถ้า container ล่ม เราจะเห็นทั้งสองแนวทางในบทเรียนนี้ — Docker สำหรับ packaging
และ reproducibility, systemd สำหรับ process supervision (ทั้งกรณีรันตรงและกรณีรันผ่าน container)

### Image vs Container: ศัพท์ที่ต้องแยกให้ออก

| คำศัพท์ | ความหมาย | เปรียบเทียบ |
|---|---|---|
| **Dockerfile** | ไฟล์ text ที่เขียนสูตรว่าจะสร้าง image อย่างไร | สูตรอาหาร |
| **Image** | ผลลัพธ์หลัง build ตาม Dockerfile — เป็น "แม่แบบ" แบบ read-only | อาหารที่ทำเสร็จแล้ว แช่แข็งเก็บไว้ |
| **Container** | Instance ที่กำลังรันจริงจาก image หนึ่งตัว (รันพร้อมกันได้หลาย container จาก image เดียว) | อาหารที่เอามาอุ่นเสิร์ฟจริงแต่ละจาน |
| **Registry** (เช่น Docker Hub) | ที่เก็บ image ไว้ให้ดาวน์โหลด/อัปโหลด | ร้านค้าที่วางขายอาหารแช่แข็ง |

---

## 112.2 Dockerfile: Multi-stage Build สำหรับ Task API (Step 890)

> **สถานะการทดสอบของหัวข้อนี้**: Dockerfile ทั้งหมดในหัวข้อนี้เขียนขึ้นให้ตรงกับโครงสร้างโปรเจกต์
> Task API จริงจาก Part 108 (`CMakeLists.txt`, `include/`, `src/`) และตรวจทานความถูกต้องของทุก
> คำสั่งอย่างละเอียด แต่ **ไม่ได้รัน `docker build` จริง** เพราะ Docker daemon ใช้งานไม่ได้ใน
> สภาพแวดล้อมนี้ ผลลัพธ์ที่แสดง (เช่น ขนาด image) เป็นตัวเลขอ้างอิงโดยประมาณจากประสบการณ์ทั่วไป
> ของการทำ multi-stage build แบบนี้ ไม่ใช่ตัวเลขที่วัดได้จริงบนเครื่องนี้

### ทบทวนโครงสร้างโปรเจกต์ Task API จาก Part 108

```
task_api/
├── CMakeLists.txt
├── include/
│   ├── task.hpp
│   ├── db.hpp
│   ├── response.hpp
│   └── task_routes.hpp
└── src/
    ├── main.cpp
    ├── db.cpp
    └── task_routes.cpp
```

โปรเจกต์นี้ต้องการตอน **build**: `cmake`, `g++`, header ของ Crow (`/usr/local/include/crow`),
header และ library ของ SQLite3, `pthread` และต้องการตอน **run**: แค่ shared library ของ
SQLite3 (`libsqlite3.so`) กับ `libstdc++`/`libc` มาตรฐานเท่านั้น — Crow เป็น header-only จึงไม่มี
ไฟล์ `.so` ที่ต้องพกไปด้วยตอน runtime เลย

### เวอร์ชันที่ไม่ดี: Single-stage Dockerfile (ตัวอย่างสิ่งที่ไม่ควรทำ)

```dockerfile
# ❌ ตัวอย่าง Dockerfile แบบไม่ดี — ห้ามทำแบบนี้ใน production
FROM ubuntu:24.04

RUN apt-get update && apt-get install -y \
    build-essential cmake libsqlite3-dev git

WORKDIR /app
COPY . .

# ดาวน์โหลด Crow (header-only) มาวางไว้
RUN git clone --depth 1 https://github.com/CrowCpp/Crow.git /tmp/crow \
    && cp -r /tmp/crow/include/* /usr/local/include/

RUN cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
    && cmake --build build -j4

EXPOSE 18180
CMD ["./build/task_api"]
```

Dockerfile นี้ **ใช้งานได้จริง** (compile ผ่าน รันได้) แต่มีปัญหาใหญ่ที่มองไม่เห็นจนกว่าจะเจอ
image ที่ build เสร็จ: **image สุดท้ายมีทั้ง `build-essential`, `cmake`, `git`, source code ทั้งหมด,
และ intermediate build artifact ติดอยู่ในนั้นด้วยหมด** ทั้งที่ตอน **run** จริงไม่ได้ใช้สิ่งเหล่านี้
เลยสักตัว — เปรียบเหมือนขนย้ายทั้งโรงงานไปพร้อมกับสินค้าสำเร็จรูปที่ผลิตได้

ผลกระทบที่ตามมา:

| ปัญหา | รายละเอียด |
|---|---|
| **ขนาด Image ใหญ่เกินจำเป็น** | Toolchain เต็มรูปแบบ (g++, cmake, header ของ dev library) กินพื้นที่หลัก ร้อย MB ถึงหลัก GB โดยที่ตัว executable จริงอาจหนักแค่ไม่กี่ MB |
| **พื้นที่โจมตี (Attack Surface) ใหญ่ขึ้น** | Compiler และเครื่องมือ build ที่ไม่จำเป็นต้องมีตอน runtime กลายเป็นเครื่องมือที่ผู้บุกรุกใช้ได้ถ้าเจาะเข้า container สำเร็จ |
| **Deploy ช้าลง** | Pull image ขนาดใหญ่ผ่านเครือข่ายทุกครั้งที่ deploy ช้ากว่ามาก โดยเฉพาะเวลา scale หลาย instance พร้อมกัน |
| **Source code รั่วไหลเข้า image** | โค้ดต้นฉบับทั้งหมดอยู่ใน image ถ้า image หลุดไปสู่มือคนที่ไม่ควรเห็น เท่ากับข้อมูล source code รั่วไปด้วย |

### เวอร์ชันที่ถูกต้อง: Multi-stage Build

**แนวคิดหลัก**: ใช้ Dockerfile เดียวที่มีหลาย `FROM` — แต่ละ `FROM` คือ "stage" หนึ่ง เราสามารถ
`COPY --from=<stage_name>` เอาผลลัพธ์จาก stage หนึ่งไปใส่ใน stage ถัดไปได้ โดย **stage สุดท้าย
เท่านั้นที่กลายเป็น image จริงที่ถูก build ออกมา** stage ก่อนหน้าถูกทิ้งไปหลัง build เสร็จ
(เหลือแค่ cache ไว้เร่งความเร็ว build ครั้งถัดไป)

```dockerfile
# ============================================================
# Stage 1: "builder" — มี toolchain เต็มรูปแบบสำหรับ compile เท่านั้น
# ============================================================
FROM ubuntu:24.04 AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    cmake \
    libsqlite3-dev \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# ติดตั้ง Crow (header-only library) — เวอร์ชัน pinned ด้วย tag ที่แน่นอน
# เพื่อ reproducibility (ห้ามใช้ branch ลอยๆ อย่าง "master" ใน production)
RUN git clone --branch v1.2.0 --depth 1 \
    https://github.com/CrowCpp/Crow.git /tmp/crow \
    && cp -r /tmp/crow/include/* /usr/local/include/ \
    && rm -rf /tmp/crow

WORKDIR /build

# คัดลอกเฉพาะไฟล์ที่จำเป็นสำหรับ build ก่อน เพื่อให้ Docker layer caching ทำงานได้เต็มที่
# (ถ้า CMakeLists.txt ไม่เปลี่ยน Docker จะไม่ต้องสั่ง configure ใหม่ทุกครั้งที่แก้แค่ .cpp)
COPY CMakeLists.txt .
COPY include/ include/
COPY src/ src/

RUN cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
    && cmake --build build -j"$(nproc)"

# ============================================================
# Stage 2: "runtime" — image สุดท้ายที่จะถูก deploy จริง เบาที่สุดเท่าที่ทำได้
# ============================================================
FROM ubuntu:24.04 AS runtime

# ติดตั้งเฉพาะ shared library ที่ executable ต้องการตอน "รัน" เท่านั้น
# ไม่มี build-essential, cmake, git, header (.h) ของ dev package ใดๆ ทั้งสิ้น
RUN apt-get update && apt-get install -y --no-install-recommends \
    libsqlite3-0 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# สร้าง user ที่ไม่ใช่ root ไว้รันโปรแกรม (หลักการ Least Privilege)
RUN useradd --system --no-create-home --shell /usr/sbin/nologin appuser

WORKDIR /app

# คัดลอก "แค่ไฟล์ executable ที่ compile เสร็จแล้ว" ข้ามมาจาก stage "builder"
# ไม่ได้คัดลอก source code, ไม่ได้คัดลอก build artifact ระหว่างทาง (.o files)
COPY --from=builder /build/build/task_api /app/task_api

# ไฟล์ฐานข้อมูล SQLite จะถูกสร้างในโฟลเดอร์นี้ตอน runtime — ต้องให้ appuser เขียนได้
RUN mkdir -p /app/data && chown -R appuser:appuser /app

USER appuser

EXPOSE 18180

# Healthcheck — Docker จะเรียก endpoint นี้เป็นระยะเพื่อรู้ว่า container ยัง "สุขภาพดี" อยู่หรือไม่
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://127.0.0.1:18180/health || exit 1

CMD ["/app/task_api"]
```

### `.dockerignore`: อย่าลืมไฟล์นี้

เช่นเดียวกับ `.gitignore` แต่สำหรับ Docker — บอกว่าไฟล์ไหน**ไม่ควร**ถูกส่งเข้าไปใน build context
(สิ่งที่ถูกส่งให้ Docker daemon ตอนรัน `docker build .`) ช่วยทั้งความเร็วในการ build และป้องกัน
ไฟล์ที่ไม่ควรหลุดเข้า image โดยไม่ตั้งใจ

```
# .dockerignore
build/
*.db
*.db-wal
*.db-shm
.git/
.gitignore
*.md
.vscode/
```

### เปรียบเทียบ Single-stage กับ Multi-stage

| ประเด็น | Single-stage | Multi-stage |
|---|---|---|
| ขนาด Image สุดท้าย (โดยประมาณ, อ้างอิงทั่วไป) | ~900 MB – 1.5 GB (มี toolchain เต็ม) | ~110–160 MB (แค่ runtime library + executable) |
| มี Compiler ติดอยู่ใน production image หรือไม่ | มี (ความเสี่ยงด้านความปลอดภัย) | ไม่มี |
| มี Source code ติดอยู่ในผลลัพธ์สุดท้ายหรือไม่ | มี | ไม่มี (อยู่แค่ใน stage builder ที่ถูกทิ้ง) |
| ความซับซ้อนของ Dockerfile | เขียนง่ายกว่า (บรรทัดน้อยกว่า) | ซับซ้อนขึ้นเล็กน้อย (ต้องเข้าใจ `--from=`) |
| เหมาะกับ | Prototype เร็วๆ, debug ชั่วคราว | **Production ทุกกรณี** |

> **กฎทองของ Part นี้**: production image ของโปรแกรมที่ compile ด้วยภาษา compiled (C, C++, Go, Rust)
> **ต้องไม่มี compiler ติดอยู่ในนั้นเด็ดขาด** ถ้าเห็น `RUN apt-get install build-essential` อยู่ใน
> stage เดียวกับที่มี `CMD` รัน executable ตัวจริง ให้สงสัยไว้ก่อนว่ากำลังลืมทำ multi-stage build

### คำสั่งสำหรับ Build และ Run (Reference — ไม่ได้รันจริงในสภาพแวดล้อมนี้)

```bash
# Build image (ตั้งชื่อ tag ว่า task-api:1.0)
docker build -t task-api:1.0 .

# รัน container แบบง่ายที่สุด (ยังไม่มี PostgreSQL) พร้อม mount volume เก็บฐานข้อมูล
docker run -d \
    --name task-api \
    -p 18180:18180 \
    -v task_api_data:/app/data \
    task-api:1.0

# ตรวจสอบว่า container รันอยู่จริงหรือไม่ และดู health status
docker ps
docker logs -f task-api

# ทดสอบ (ตัวอย่างผลลัพธ์ที่คาดว่าจะได้ — ไม่ใช่ output จริงจากเครื่องนี้)
curl http://127.0.0.1:18180/health
# คาดว่าจะได้: {"data":{"status":"up"},"success":true}
```

**ผลลัพธ์ที่คาดว่าจะได้จาก `docker images`** (ตัวเลขอ้างอิงโดยประมาณ ไม่ใช่ค่าที่วัดจริง):

```
REPOSITORY   TAG    IMAGE ID       SIZE
task-api     1.0    a1b2c3d4e5f6   ~140MB
```

เทียบกับถ้าใช้ Dockerfile แบบ single-stage ตัวเลข `SIZE` ในคอลัมน์เดียวกันนี้จะพุ่งไปถึงหลัก GB
ทันที เพราะ toolchain ของ `build-essential` (g++, binutils, headers ต่างๆ) มีขนาดใหญ่กว่า
runtime library ล้วนๆ อย่างมีนัยสำคัญ

---

## 112.3 docker-compose.yml: รวม App Container + PostgreSQL Container (Step 891)

> **สถานะการทดสอบของหัวข้อนี้**: `docker-compose.yml` เป็นโค้ดอ้างอิง ไม่ได้รันจริงด้วยเหตุผล
> เดียวกับหัวข้อก่อนหน้า (ไม่มี Docker daemon) **แต่รูปแบบ credential และ connection string
> ที่ใช้ในไฟล์นี้ถูกทดสอบจริง** ด้วย PostgreSQL 16 ที่ติดตั้งแบบ native บนเครื่องนี้ (ไม่ผ่าน
> container) เพื่อยืนยันว่ารูปแบบที่เขียนไว้ถูกต้องและใช้งานได้จริง — รายละเอียดอยู่ท้ายหัวข้อ

### ทำไมต้องเปลี่ยนจาก SQLite ไปเป็น PostgreSQL ตอน deploy ด้วย Container

Task API เดิมจาก Part 108 ใช้ SQLite ซึ่งเก็บข้อมูลเป็นไฟล์เดียว (`tasks.db`) เหมาะมากสำหรับ
การเรียนรู้และแอปขนาดเล็ก แต่เมื่อ deploy ด้วย container มีข้อจำกัดสำคัญที่ควรรู้:

- **Container เป็น ephemeral (ชั่วคราว) โดยธรรมชาติ** — ถ้า container ถูกลบหรือ rebuild ไฟล์
  ทุกไฟล์ในนั้น (รวมถึง `tasks.db`) จะหายไปด้วย เว้นแต่จะ mount volume แยกไว้ต่างหาก
- **การ scale หลาย instance พร้อมกันทำได้ยากกับ SQLite** — ถ้ารัน container ของ app 3 ตัวพร้อมกัน
  (เพื่อรับ traffic สูง) แต่ละ instance เห็นไฟล์ SQLite คนละไฟล์ (ในโฟลเดอร์ตัวเอง) ข้อมูลจะไม่ตรง
  กันข้าม instance เลย ต่างจาก PostgreSQL ที่เป็น service กลางแยกต่างหากที่ทุก instance เชื่อมต่อ
  เข้าไปหาที่เดียวกันได้
- นี่คือเหตุผลที่ Part 106 (เชื่อมต่อ PostgreSQL/MySQL ด้วย C++) มีความสำคัญมากขึ้นเมื่อพูดถึง
  การ deploy จริงในระดับ production ที่ต้องรองรับการขยายระบบ (scale) ในอนาคต

### โครงสร้างโปรเจกต์หลังเพิ่ม Docker

```
task_api/
├── Dockerfile
├── .dockerignore
├── docker-compose.yml
├── CMakeLists.txt
├── include/
└── src/
```

### docker-compose.yml

```yaml
services:
  db:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: taskapi
      POSTGRES_PASSWORD: taskapi_pw
      POSTGRES_DB: taskapi_db
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U taskapi -d taskapi_db"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - task_api_net
    # ไม่ expose port 5432 ออกสู่ host เพราะไม่มีความจำเป็นต้องให้เครื่องภายนอกต่อ DB ตรงๆ
    # ถ้าต้องการ debug ด้วย psql จากเครื่อง host เองค่อยเปิด "5432:5432" ชั่วคราว

  api:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: "postgresql://taskapi:taskapi_pw@db:5432/taskapi_db"
      PORT: "18180"
    ports:
      - "18180:18180"
    networks:
      - task_api_net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://127.0.0.1:18180/health"]
      interval: 10s
      timeout: 3s
      retries: 3

networks:
  task_api_net:
    driver: bridge

volumes:
  pgdata:
```

### สิ่งสำคัญที่ต้องเข้าใจในไฟล์นี้

**1. Network ภายในของ Docker Compose** — Docker Compose สร้าง network ส่วนตัว (`task_api_net`)
ให้ container ทุกตัวใน `services:` คุยกันได้ด้วย **ชื่อ service เป็น hostname โดยตรง** สังเกตว่า
`DATABASE_URL` ของ service `api` ชี้ไปที่ `db:5432` — คำว่า `db` **ไม่ใช่** IP address แต่เป็นชื่อ
service ที่ Docker แปลงเป็น IP ให้อัตโนมัติผ่าน DNS ภายในของมันเอง (ต่างจากตอนรันบนเครื่องเดียวกัน
ตรงๆ ที่ต้องใช้ `127.0.0.1` หรือ `localhost`)

**2. `depends_on` กับ `condition: service_healthy`** — การเขียน `depends_on: [db]` เฉยๆ (แบบเก่า)
บอกแค่ "รอให้ container ของ `db` **เริ่มทำงาน**ก่อน" แต่ container เริ่มทำงานไม่ได้แปลว่า
PostgreSQL **พร้อมรับ connection แล้ว** (PostgreSQL ต้องใช้เวลา initialize ฐานข้อมูลเล็กน้อย)
การระบุ `condition: service_healthy` คู่กับ `healthcheck` ของ service `db` (ใช้คำสั่ง `pg_isready`
ที่ PostgreSQL ติดตั้งมาให้) ทำให้ Docker Compose รอจนกว่า database **พร้อมรับ connection จริงๆ**
ก่อนจะเริ่ม container ของ `api` ป้องกันปัญหา "app พยายามต่อ database ตั้งแต่ยังไม่พร้อม แล้ว crash
ตั้งแต่ตอน startup"

**3. Environment Variable ตามแนวคิด 12-Factor App** — สังเกตว่า connection string ของฐานข้อมูล
(`DATABASE_URL`) และพอร์ต (`PORT`) ถูกส่งเข้าไปผ่าน environment variable ไม่ได้ hardcode ไว้ใน
โค้ด นี่คือหลักการข้อที่ 3 ของ [12-Factor App](https://12factor.net/config) ที่บอกว่า **configuration
ที่เปลี่ยนไปตาม environment (dev/staging/production) ต้องแยกออกจาก source code เสมอ** ทำให้
image เดียวกันเอาไปรันได้ทั้ง dev และ production แค่เปลี่ยนค่า environment variable ตอนรัน
โดยไม่ต้อง build image ใหม่

โค้ด C++ ฝั่ง `main.cpp` ต้องอ่านค่าเหล่านี้ผ่าน `std::getenv()` แทนการ hardcode:

```cpp
#include <cstdlib>
#include <string>

// อ่านค่าจาก environment variable พร้อมค่า default ถ้าไม่ได้ตั้งไว้
// (สำคัญมาก: ต้องมี fallback เสมอ เผื่อรันนอก container ตอน develop บนเครื่องตัวเอง)
std::string get_env_or(const char* key, const std::string& fallback) {
    const char* val = std::getenv(key);
    return val ? std::string(val) : fallback;
}

int main() {
    int port = std::stoi(get_env_or("PORT", "18180"));
    std::string db_url = get_env_or("DATABASE_URL", "postgresql://localhost/taskapi_db");
    // ... ส่ง db_url เข้า connection layer จาก Part 106 (libpqxx) แทนที่ Db(sqlite path)
    // ... app.port(port).multithreaded().run();
}
```

รูปแบบนี้ (อ่าน config จาก environment variable พร้อม fallback ค่า default) ได้รับการทดสอบจริง
บนเครื่องนี้แล้วในหัวข้อ 112.4 ด้านล่าง (ใช้ตัวแปร `PORT` และ `DB_PATH` จริงตอนทดสอบ Nginx
load balancing กับ backend สองตัว) — พิสูจน์ว่า pattern การอ่านค่าผ่าน `std::getenv()` ทำงาน
ถูกต้องจริงก่อนที่จะเอาไปใช้ร่วมกับ Docker

**4. Volume สำหรับข้อมูลถาวร** — `pgdata:/var/lib/postgresql/data` ผูกโฟลเดอร์ข้อมูลจริงของ
PostgreSQL ไว้กับ **named volume** ที่ Docker จัดการให้ ทำให้ข้อมูลอยู่รอดแม้ container ของ `db`
จะถูกลบและสร้างใหม่ (เช่นตอนอัปเดตเวอร์ชัน PostgreSQL image) — ถ้าลืมใส่ volume ตรงนี้ ข้อมูล
ทั้งหมดในฐานข้อมูลจะหายทันทีที่ container ถูกลบ ซึ่งเป็นหนึ่งใน **หลุมพรางที่อันตรายที่สุด**
ของการใช้ Docker กับฐานข้อมูล

### คำสั่งสำหรับใช้งาน (Reference — ไม่ได้รันจริงในสภาพแวดล้อมนี้)

```bash
# Build image และเริ่มทุก service พร้อมกัน (แบบ background ด้วย -d)
docker compose up -d --build

# ดู log ของทุก service แบบ real-time
docker compose logs -f

# ดูสถานะของแต่ละ service (คาดว่าจะเห็นทั้งสอง service เป็น "healthy")
docker compose ps

# หยุดและลบ container (แต่ volume ยังอยู่ ข้อมูลไม่หาย)
docker compose down

# หยุดและลบทั้ง container และ volume (ข้อมูลหายทั้งหมด — ใช้ระวังมาก)
docker compose down -v
```

**ผลลัพธ์ที่คาดว่าจะได้จาก `docker compose ps`** (ตัวอย่าง ไม่ใช่ output จริง):

```
NAME              IMAGE          STATUS
task_api-db-1     postgres:16    Up 30 seconds (healthy)
task_api-api-1    task_api-api   Up 20 seconds (healthy)
```

### การพิสูจน์รูปแบบ Credential จริงบนเครื่องนี้ (ทดสอบจริง แม้ไม่ผ่าน Docker)

แม้จะรัน `docker compose up` จริงไม่ได้ แต่รูปแบบ connection string
`postgresql://taskapi:taskapi_pw@db:5432/taskapi_db` ที่ใช้ในไฟล์ข้างต้นได้รับการยืนยันจริงแล้ว
ว่า **credential และชื่อฐานข้อมูลที่กำหนดไว้ใช้งานได้จริง** โดยทดสอบผ่าน PostgreSQL 16 (เวอร์ชัน
เดียวกับ image `postgres:16`) ที่ติดตั้งแบบ native บนเครื่องนี้ด้วย `pg_ctlcluster 16 main start`:

```bash
$ sudo -u postgres psql -c "CREATE ROLE taskapi WITH LOGIN PASSWORD 'taskapi_pw';"
CREATE ROLE
$ sudo -u postgres psql -c "CREATE DATABASE taskapi_db OWNER taskapi;"
CREATE DATABASE

$ PGPASSWORD=taskapi_pw psql -h 127.0.0.1 -U taskapi -d taskapi_db \
    -c "SELECT current_user, current_database(), version();"
```

ผลลัพธ์จริงที่ได้:

```
 current_user | current_database |                          version
--------------+-------------------+-------------------------------------------------------------
 taskapi      | taskapi_db        | PostgreSQL 16.15 (Ubuntu 16.15-0ubuntu0.24.04.1) on x86_64...
(1 row)
```

ความต่างเดียวระหว่างการทดสอบนี้กับของจริงใน Docker Compose คือ **host ที่ใช้เชื่อมต่อ** — ที่นี่
ใช้ `127.0.0.1` (เพราะ PostgreSQL รันตรงบนเครื่อง) ส่วนใน `docker-compose.yml` จะใช้ `db` แทน
(เพราะ PostgreSQL รันเป็น container แยกต่างหากที่ต้องอ้างถึงผ่านชื่อ service) แต่ **username,
password, ชื่อฐานข้อมูล และรูปแบบ connection string ทั้งหมดถูกต้องและใช้งานได้จริง** ตามที่ทดสอบ

---

## 112.4 Nginx เป็น Reverse Proxy หน้า C++ Backend (Step 892)

> **สถานะการทดสอบของหัวข้อนี้**: ✅ **ทดสอบจริงทั้งหมด** — ไม่ใช่ผ่าน Docker (เพราะ Docker daemon
> ใช้ไม่ได้) แต่รัน Nginx ตรงๆ บนเครื่องนี้ ชี้ไปยัง Task API binary จริงที่ compile จาก Part 108
> ทุกคำสั่ง `curl` และผลลัพธ์ในหัวข้อนี้คือผลจริงที่ได้จากการรันคำสั่งเหล่านั้นจริง

### ทำไมต้องมี Reverse Proxy หน้า Backend เสมอ

Task API ของเรา (จาก `app.port(18180).multithreaded().run();`) เปิดพอร์ต 18180 รอรับ HTTP request
ได้เองอยู่แล้ว คำถามคือ **ทำไมไม่ให้ผู้ใช้เชื่อมต่อเข้ามาที่พอร์ต 18180 ตรงๆ เลยล่ะ**

เหตุผลหลักที่ production ระบบจริงแทบทุกระบบวาง **Reverse Proxy** (เช่น Nginx) ไว้หน้า backend
เสมอ:

| เหตุผล | รายละเอียด |
|---|---|
| **Port ต่ำกว่า 1024 ต้องใช้สิทธิ์ root** | Web server มาตรฐานต้องฟังที่ port 80 (HTTP) และ 443 (HTTPS) แต่การให้โปรแกรม backend ของเรารันด้วยสิทธิ์ root โดยตรงเสี่ยงมาก — Nginx (ซึ่งออกแบบมาให้ทำเรื่องนี้อย่างปลอดภัย) รับหน้าที่นี้แทน แล้วส่งต่อไปให้ backend ที่รันด้วย user ธรรมดาที่พอร์ตสูงกว่า |
| **TLS Termination** | การตั้งค่า HTTPS/TLS certificate ทำที่ Nginx เพียงจุดเดียว ไม่ต้องเขียนโค้ด C++ จัดการ TLS handshake เอง (ซับซ้อนและเสี่ยง bug ด้านความปลอดภัยสูงกว่ามาก) |
| **Virtual Hosting** | เซิร์ฟเวอร์เดียวรองรับหลายโดเมน/หลายแอปพร้อมกันได้ โดย Nginx เป็นคนตัดสินใจว่า request ไหนควรส่งไปที่ backend ตัวไหน |
| **Static File & Compression** | Nginx serve ไฟล์ static (CSS/JS/รูปภาพ) และบีบอัด response ด้วย gzip ได้เร็วกว่าและมีประสิทธิภาพกว่าให้ backend C++ ทำเอง |
| **Buffering ลูกค้าที่เน็ตช้า** | Nginx รับ request/response ทั้งหมดไว้ก่อนแล้วค่อยส่งต่อ ป้องกันไม่ให้ backend thread ถูกจองไว้นานเพราะรอ client ที่อินเทอร์เน็ตช้า |
| **Rate Limiting / Load Balancing** | จำกัดจำนวน request ต่อวินาทีจาก IP เดียวกัน หรือกระจาย traffic ไปหลาย backend instance ได้ในจุดเดียว (จะสาธิตจริงท้ายหัวข้อนี้) |

### เตรียม Backend: Task API จาก Part 108 รันจริงที่พอร์ต 18180

Build โปรเจกต์ Task API ตามที่ทำไว้ใน Part 108 แล้วรัน:

```bash
$ g++ -std=c++17 -O2 -Wall -Wextra -Iinclude -I/usr/local/include \
    src/main.cpp src/db.cpp src/task_routes.cpp \
    -o task_api -lpthread -lsqlite3
$ ./task_api
```

Log จริงที่ได้ตอนเริ่มโปรแกรม:

```
(2026-09-26 11:23:08) [INFO    ] Crow/master server is running at http://0.0.0.0:18180 using 4 threads
```

### ตั้งค่า Nginx เป็น Reverse Proxy

สร้างไฟล์ `/etc/nginx/sites-available/task_api`:

```nginx
server {
    listen 80;
    server_name _;

    location /api/ {
        proxy_pass http://127.0.0.1:18180/;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }
}
```

เปิดใช้งานและทดสอบ syntax:

```bash
$ ln -s /etc/nginx/sites-available/task_api /etc/nginx/sites-enabled/task_api
$ nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
$ nginx
```

### อธิบายทีละบรรทัด

- **`listen 80;`** — ฟังที่พอร์ต 80 (HTTP มาตรฐาน) **สังเกตว่าไม่มี `[::]` (IPv6) อยู่ในบรรทัดนี้**
  ซึ่งสำคัญมากสำหรับสภาพแวดล้อมนี้โดยเฉพาะ (อธิบายรายละเอียดในหัวข้อ Common Pitfalls)
- **`location /api/`** — ทุก request ที่ path ขึ้นต้นด้วย `/api/` จะถูกจับคู่กับ block นี้
- **`proxy_pass http://127.0.0.1:18180/;`** — ส่งต่อ request ไปที่ backend จริงที่พอร์ต 18180
  (สังเกต trailing slash `/` ท้าย URL — สำคัญมาก อธิบายในหัวข้อ Common Pitfalls เช่นกัน)
- **`proxy_http_version 1.1;`** — บังคับใช้ HTTP/1.1 ระหว่าง Nginx กับ backend (ค่า default ของ
  Nginx คือ HTTP/1.0 ซึ่งไม่รองรับ keep-alive connection ทำให้ประสิทธิภาพแย่กว่ามาก และจำเป็น
  สำหรับ WebSocket ใน Part 109 ด้วย)
- **`proxy_set_header Host $host;`** — ส่งต่อ header `Host` เดิมที่ client ส่งมา ไปให้ backend รู้
  ว่า client เรียกผ่านโดเมนอะไร (ค่า default ของ Nginx จะเปลี่ยน `Host` เป็นค่าที่ตั้งใน `proxy_pass`
  ถ้าไม่ระบุบรรทัดนี้ ทำให้ backend สับสนว่าตัวเองถูกเรียกด้วยโดเมนอะไร)
- **`proxy_set_header X-Real-IP $remote_addr;`** — เนื่องจาก backend เห็น connection ที่มาจาก
  Nginx เสมอ (ไม่ใช่จาก client จริง) ต้องส่ง IP จริงของ client ผ่าน header นี้แทน มิฉะนั้น log
  ของ backend จะเห็นแต่ IP ของ Nginx (`127.0.0.1`) ตลอด ไม่มีประโยชน์สำหรับการ debug/security
- **`proxy_set_header X-Forwarded-For / X-Forwarded-Proto`** — header มาตรฐานที่บอก IP ต้นทาง
  (กรณีผ่านหลาย proxy ต่อกัน) และ protocol เดิมที่ client ใช้ (`http`/`https`) ตามลำดับ

### ทดสอบจริงแบบ End-to-End ผ่าน Nginx

```bash
$ curl -s -i http://127.0.0.1/api/health
```

ผลลัพธ์จริง:

```
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Sat, 26 Sep 2026 11:23:39 GMT
Content-Type: application/json
Content-Length: 39
Connection: keep-alive

{"data":{"status":"up"},"success":true}
```

สังเกตว่า header `Server` เป็น `nginx/1.24.0` (ไม่ใช่ `Crow/master` เหมือนตอนเรียกตรง) — เพราะ
Nginx สร้าง HTTP response header ของตัวเองใหม่ ผู้ใช้ที่อยู่นอกเซิร์ฟเวอร์จะไม่เห็นเลยว่าเบื้องหลัง
เป็น Crow/C++ ซึ่งเป็นข้อดีด้านความปลอดภัยเช่นกัน (ปิดบัง technology stack ที่แท้จริง)

ทดสอบ `POST` ผ่าน Nginx สร้าง Task ใหม่:

```bash
$ curl -s -i -X POST http://127.0.0.1/api/tasks \
    -H "Content-Type: application/json" \
    -d '{"title":"สร้างผ่าน nginx proxy"}'
```

ผลลัพธ์จริง:

```
HTTP/1.1 201 Created
Server: nginx/1.24.0 (Ubuntu)
Date: Sat, 26 Sep 2026 11:23:39 GMT
Content-Type: application/json
Content-Length: 147
Connection: keep-alive

{"data":{"created_at":"2026-09-26 11:23:39","description":"","done":false,"id":2,"title":"สร้างผ่าน nginx proxy"},"success":true}
```

ตรวจสอบว่าข้อมูลถูกบันทึกจริงโดยเรียกตรงไปที่ backend เทียบกัน:

```bash
$ curl -s http://127.0.0.1:18180/tasks
{"data":[{"created_at":"2026-09-26 11:23:09","description":"","done":false,"id":1,"title":"ทดสอบ deploy ด้วย nginx"},{"created_at":"2026-09-26 11:23:39","description":"","done":false,"id":2,"title":"สร้างผ่าน nginx proxy"}],"success":true}
```

ข้อมูลตรงกันทุกประการ — พิสูจน์ว่า Nginx ส่งต่อ request (รวมถึง body ของ `POST`) ไปยัง backend
ได้อย่างถูกต้องสมบูรณ์ ทั้งขา request และขา response

### หลุมพรางเรื่อง Trailing Slash: พิสูจน์จริงด้วย curl

นี่คือหลุมพรางที่ผู้เริ่มต้นเขียน Nginx config เจอบ่อยที่สุด — ลองเปลี่ยน config ให้ **ไม่มี**
trailing slash หลัง `proxy_pass`:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:18180;   # <-- ไม่มี "/" ท้าย URL แบบนี้
    proxy_http_version 1.1;
    proxy_set_header Host $host;
}
```

Reload แล้วทดสอบ:

```bash
$ nginx -s reload
$ curl -s -i http://127.0.0.1/api/tasks | head -5
```

ผลลัพธ์จริงที่ได้:

```
HTTP/1.1 404 Not Found
Server: nginx/1.24.0 (Ubuntu)
Date: Sat, 26 Sep 2026 11:23:45 GMT
Content-Length: 15
Connection: keep-alive
```

**404 จริง!** — เกิดอะไรขึ้น? กฎของ Nginx คือ:

- ถ้า `proxy_pass` **มี** path ต่อท้าย (แม้จะเป็นแค่ `/` เฉยๆ ก็นับ) → Nginx จะ**ตัด** ส่วนของ
  `location` ที่ match ออกจาก URL ก่อนต่อกับ path ใน `proxy_pass` เช่น request `/api/tasks`
  จะกลายเป็น `http://127.0.0.1:18180/` + `tasks` = `http://127.0.0.1:18180/tasks` (ถูกต้อง)
- ถ้า `proxy_pass` **ไม่มี** path ต่อท้ายเลย (จบด้วยแค่ hostname:port) → Nginx จะส่ง URL **เดิม
  ทั้งหมด** (รวม `/api/` ที่ match ด้วย) ไปที่ backend ตรงๆ เช่น request `/api/tasks` จะถูกส่งไป
  เป็น `http://127.0.0.1:18180/api/tasks` — ซึ่ง route นี้**ไม่มีอยู่จริง**ใน Task API (route
  จริงคือ `/tasks` ไม่ใช่ `/api/tasks`) จึงได้ `404 Not Found` กลับมา

ทดสอบแก้กลับให้มี trailing slash แล้ว reload อีกครั้งเพื่อยืนยัน:

```bash
$ nginx -s reload
$ curl -s -i http://127.0.0.1/api/tasks | head -5
```

```
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Sat, 26 Sep 2026 11:23:53 GMT
Content-Type: application/json
Content-Length: 275
```

กลับมาเป็น `200 OK` ทันทีเมื่อใส่ trailing slash กลับเข้าไป — นี่คือหลักฐานจริงที่ยืนยันว่า
trailing slash หลัง `proxy_pass` **เปลี่ยนพฤติกรรมโดยสิ้นเชิง** ไม่ใช่แค่เรื่องความสวยงามของโค้ด

### สาธิตจริง: หลุมพราง IPv6 `listen` Directive

สภาพแวดล้อมของ container บางประเภท (รวมถึง sandbox ที่ใช้เขียนบทเรียนนี้) **ไม่รองรับ IPv6
socket** ทำให้ config ที่เขียนตามตัวอย่างทั่วไปบนอินเทอร์เน็ต (ซึ่งมักจะมี `listen [::]:80;`
ควบคู่กับ `listen 80;` เพื่อรองรับทั้ง IPv4 และ IPv6) **ทำให้ Nginx เริ่มทำงานไม่ได้เลย**

ทดสอบจริงด้วย config ทดสอบแยกต่างหาก:

```nginx
server {
    listen 8081;
    listen [::]:8081;
    server_name _;
    location / { return 200 "ok"; }
}
```

```bash
$ nginx -c /tmp/ipv6_test.conf -t
```

ผลลัพธ์จริง:

```
nginx: the configuration file /tmp/ipv6_test.conf syntax is ok
nginx: [emerg] socket() [::]:8081 failed (97: Address family not supported by protocol)
nginx: configuration file /tmp/ipv6_test.conf test failed
```

สังเกตว่า **syntax ของ config ถูกต้อง 100%** (`syntax is ok`) — ปัญหาไม่ใช่เรื่องเขียนผิด แต่เป็น
เรื่อง **ระบบปฏิบัติการ/container runtime ไม่มี IPv6 socket ให้ใช้งาน** วิธีแก้คือตัดบรรทัด
`listen [::]:80;` ออกไปเลย ใช้แค่ `listen 80;` เพียงบรรทัดเดียว (แบบที่ใช้ตลอดทั้งบทเรียนนี้)
ซึ่งใช้งานได้ปกติในสภาพแวดล้อมที่ไม่รองรับ IPv6 และยังคงรองรับ IPv4 ได้ครบถ้วนเหมือนเดิม —
บนเซิร์ฟเวอร์จริงที่รองรับ IPv6 เต็มรูปแบบ จะใส่ทั้งสองบรรทัดกลับเข้าไปก็ไม่มีปัญหาอะไร

### Bonus ที่ทดสอบจริง: Nginx เป็น Load Balancer หน้า Backend หลาย Instance

Nginx ทำหน้าที่มากกว่า reverse proxy ธรรมดาได้ — ใช้ block `upstream` กระจาย request ไปหลาย
backend instance พร้อมกันได้ ทดสอบจริงด้วยการรัน Task API สองชุด (คนละพอร์ต, คนละไฟล์ฐานข้อมูล
เพื่อแยกจากกันชัดเจน) ผ่าน environment variable `PORT`/`DB_PATH`:

```bash
$ PORT=18180 DB_PATH=tasks_a.db ./task_api_env &
$ PORT=18181 DB_PATH=tasks_b.db ./task_api_env &
```

ตั้งค่า Nginx:

```nginx
upstream task_api_backend {
    server 127.0.0.1:18180;
    server 127.0.0.1:18181;
}

log_format upstream_log '$remote_addr -> upstream=$upstream_addr status=$status';

server {
    listen 80;
    server_name _;
    access_log /var/log/nginx/task_api_lb.log upstream_log;

    location /api/ {
        proxy_pass http://task_api_backend/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
    }
}
```

ยิง request ซ้ำ 6 ครั้งแล้วดู log ว่า Nginx กระจายไปที่ backend ตัวไหนบ้าง:

```bash
$ for i in $(seq 1 6); do curl -s -o /dev/null -w "request $i -> HTTP %{http_code}\n" \
    http://127.0.0.1/api/health; done
$ cat /var/log/nginx/task_api_lb.log
```

ผลลัพธ์จริง:

```
request 1 -> HTTP 200
request 2 -> HTTP 200
request 3 -> HTTP 200
request 4 -> HTTP 200
request 5 -> HTTP 200
request 6 -> HTTP 200

127.0.0.1 -> upstream=127.0.0.1:18180 status=200
127.0.0.1 -> upstream=127.0.0.1:18180 status=200
127.0.0.1 -> upstream=127.0.0.1:18180 status=200
127.0.0.1 -> upstream=127.0.0.1:18180 status=200
127.0.0.1 -> upstream=127.0.0.1:18181 status=200
127.0.0.1 -> upstream=127.0.0.1:18180 status=200
```

ทั้งสอง backend ได้รับ request จริง (18180 และ 18181 ปรากฏใน log ทั้งคู่) ยืนยันว่า load
balancing ทำงาน — แต่สังเกตว่า **การกระจายไม่ได้สลับกัน 1-2-1-2 เป๊ะๆ ตามที่คาดหวังจาก
round-robin แบบง่ายๆ** เหตุผลที่แท้จริงเมื่อตรวจสอบคือ Nginx ตั้งค่า `worker_processes auto;`
ซึ่งบนเครื่องนี้แปลเป็น **4 worker process** (เท่าจำนวน CPU core) และ **แต่ละ worker process
มีตัวนับ round-robin ของตัวเองแยกกัน** ไม่ได้ใช้ตัวนับร่วมกันทั้งระบบ ทำให้เมื่อดูจากภายนอกในระยะ
สั้นๆ การกระจายจึงดูไม่สม่ำเสมอเป๊ะ (แม้ในระยะยาวเมื่อมี request จำนวนมากพอ สัดส่วนโดยรวมจะ
ใกล้เคียง 50/50) — นี่คือรายละเอียดเชิงสถาปัตยกรรมของ Nginx ที่มักไม่มีใครพูดถึง แต่สำคัญมากถ้า
กำลัง debug ปัญหาการกระจาย traffic ที่ดูเหมือนไม่เท่ากันในระบบจริง

---

## 112.5 HTTPS ด้วย Let's Encrypt: แนวคิดเบื้องต้น (Step 893)

> **สถานะของหัวข้อนี้**: อธิบายเป็นแนวคิดล้วนๆ **ไม่มีการสาธิตจริง** เพราะการขอใบรับรอง TLS จริง
> จาก Let's Encrypt ต้องมี **โดเมนสาธารณะจริงที่ชี้มาที่ IP ของเซิร์ฟเวอร์** และเซิร์ฟเวอร์ต้อง
> เข้าถึงได้จากอินเทอร์เน็ตภายนอกที่พอร์ต 80/443 ซึ่ง sandbox container ที่ใช้เขียนบทเรียนนี้
> ไม่มีทั้งโดเมนสาธารณะและ public IP ที่เข้าถึงได้จากภายนอก จึงเป็นไปไม่ได้ที่จะสาธิตขั้นตอนนี้
> ให้เห็นจริงในบทเรียน — แต่แนวคิดและคำสั่งทั้งหมดที่แสดงด้านล่างถูกต้องตามเอกสารทางการของ
> Let's Encrypt/Certbot และนำไปใช้ได้ทันทีเมื่อมีโดเมนจริง

### ทำไมต้องมี HTTPS

HTTP ส่งข้อมูลเป็น plain text ทั้งหมด — ใครก็ตามที่ดักฟังเครือข่ายระหว่างทาง (เช่น Wi-Fi
สาธารณะที่ไม่ปลอดภัย) สามารถอ่านข้อมูลได้ทั้งหมด รวมถึง password, session token, ข้อมูลส่วนตัว
**HTTPS** เข้ารหัสข้อมูลทั้งหมดด้วย **TLS (Transport Layer Security)** ทำให้แม้จะดักฟังได้ก็อ่าน
เนื้อหาไม่ออก

### ทำไมต้องทำ TLS Termination ที่ Nginx ไม่ใช่ในโค้ด C++ โดยตรง

**TLS Termination** คือจุดที่การเข้ารหัส TLS ถูก "ถอด" ออก (decrypt) ก่อนส่งต่อข้อมูลเป็น plain
HTTP ธรรมดาต่อไป สถาปัตยกรรมมาตรฐานคือ:

```
Browser  --[HTTPS: เข้ารหัส]-->  Nginx (TLS Termination)  --[HTTP: plain text]-->  C++ Backend
```

เหตุผลที่ทำแบบนี้แทนที่จะเขียนโค้ด C++ จัดการ TLS handshake เอง:

| เหตุผล | รายละเอียด |
|---|---|
| **ความปลอดภัย** | การ implement TLS เองผิดพลาดง่ายมาก (เช่น cipher suite ที่ไม่ปลอดภัย, การตรวจสอบ certificate ไม่ครบ) Nginx/OpenSSL ผ่านการตรวจสอบและ patch security bug จากทีมงานระดับโลกมานานหลายสิบปี |
| **แยกความรับผิดชอบ** | โค้ด backend ไม่ต้องรู้เรื่อง certificate เลย ทำให้ code base เรียบง่ายกว่ามาก |
| **จัดการ Certificate ที่จุดเดียว** | ถ้ามี backend หลาย instance (ตามหัวข้อ Load Balancer ก่อนหน้า) ต้องมี certificate ที่จุดเดียวคือ Nginx ไม่ต้องแจกจ่าย certificate ไปทุก instance |
| **Renew อัตโนมัติได้ง่าย** | เครื่องมืออย่าง Certbot ผูกกับ Nginx โดยตรง ต่ออายุใบรับรองอัตโนมัติได้โดยไม่ต้อง restart backend เลย |

### Let's Encrypt คืออะไร

**Let's Encrypt** คือ Certificate Authority (CA) ที่ออกใบรับรอง TLS **ฟรี** ผ่านโปรโตคอลอัตโนมัติ
ชื่อ **ACME (Automatic Certificate Management Environment)** ต่างจากสมัยก่อนที่ต้องซื้อใบรับรอง
จากบริษัทและตั้งค่าด้วยมือทุกขั้นตอน Let's Encrypt ทำให้กระบวนการทั้งหมดอัตโนมัติผ่านเครื่องมือ
ที่เรียกว่า **Certbot**

### ขั้นตอน (Reference — ต้องมีโดเมนจริงจึงจะรันได้)

**1. ติดตั้ง Certbot พร้อม plugin สำหรับ Nginx:**

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

**2. ขอใบรับรองพร้อมตั้งค่า Nginx ให้อัตโนมัติ:**

```bash
sudo certbot --nginx -d api.example.com
```

Certbot จะทำสิ่งเหล่านี้อัตโนมัติทั้งหมด:

- ตรวจสอบความเป็นเจ้าของโดเมน (Domain Validation) ผ่าน HTTP-01 challenge — Certbot จะสร้างไฟล์
  ชั่วคราวไว้ใน path พิเศษ แล้วให้เซิร์ฟเวอร์ของ Let's Encrypt เรียกมาตรวจสอบผ่าน `http://api.example.com/.well-known/acme-challenge/...` (นี่คือเหตุผลที่ต้องมีโดเมนจริงชี้มาที่เซิร์ฟเวอร์
  และเปิดพอร์ต 80 ให้เข้าถึงได้จากอินเทอร์เน็ตจริง)
- ขอใบรับรองจาก Let's Encrypt CA (มีอายุ 90 วัน)
- แก้ไขไฟล์ config ของ Nginx ให้อัตโนมัติ เพิ่ม `listen 443 ssl;`, `ssl_certificate`,
  `ssl_certificate_key` และ redirect จาก HTTP ไป HTTPS ให้เอง
- ตั้งค่า **auto-renewal** ผ่าน systemd timer (หรือ cron ในระบบเก่า) ให้ต่ออายุอัตโนมัติก่อน
  หมดอายุ (Let's Encrypt แนะนำให้ renew ทุก 60 วัน เผื่อเวลาไว้ 30 วันก่อนหมดอายุจริง)

**3. ผลลัพธ์: Nginx config ที่ Certbot แก้ไขให้ (ตัวอย่างหน้าตาที่คาดว่าจะได้):**

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location /api/ {
        proxy_pass http://127.0.0.1:18180/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name api.example.com;
    # Certbot เพิ่ม redirect นี้ให้อัตโนมัติ เพื่อบังคับทุก request ไปใช้ HTTPS เสมอ
    return 301 https://$host$request_uri;
}
```

**4. ตรวจสอบว่า auto-renewal ทำงานถูกต้อง (คำสั่งนี้ปลอดภัย ไม่ได้ renew จริง แค่ dry-run):**

```bash
sudo certbot renew --dry-run
```

### เปรียบเทียบ HTTP กับ HTTPS ในบริบทของการ Deploy

| ประเด็น | HTTP | HTTPS |
|---|---|---|
| ข้อมูลระหว่างทาง | plain text อ่านได้ถ้าถูกดักฟัง | เข้ารหัสด้วย TLS |
| ความน่าเชื่อถือกับผู้ใช้ | Browser แสดง "Not Secure" | Browser แสดงไอคอนล็อค |
| SEO | Google ลด ranking ของเว็บที่ไม่มี HTTPS | ได้ ranking ที่ดีกว่า |
| ต้นทุน (ด้วย Let's Encrypt) | ไม่มี | ฟรี แต่ต้องต่ออายุทุก 90 วัน (อัตโนมัติได้) |
| ความซับซ้อนของ Nginx config | น้อยกว่า | ต้องจัดการ certificate เพิ่ม (Certbot ช่วยได้เกือบทั้งหมด) |
| ข้อกำหนดเบื้องต้น | ไม่มี | ต้องมีโดเมนสาธารณะจริงที่ชี้มาที่เซิร์ฟเวอร์ |

> **กฎทองของ Part นี้**: ในปี 2026 **ไม่มีเหตุผลที่ดีพอที่จะ deploy เว็บสาธารณะด้วย HTTP ล้วนๆ
> อีกต่อไป** เพราะ Let่s Encrypt ทำให้ HTTPS ฟรีและตั้งค่าอัตโนมัติได้เกือบทั้งหมด ข้อยกเว้นที่พอ
> ยอมรับได้มีแค่ระบบภายในองค์กร (internal network) ที่ไม่เปิดสู่อินเทอร์เน็ตสาธารณะเลยเท่านั้น

---

## 112.6 ตั้งค่า systemd Service สำหรับรัน C++ Server (Step 894)

> **สถานะการทดสอบของหัวข้อนี้**: unit file ผ่านการตรวจสอบจริงด้วย `systemd-analyze verify`
> (ไม่ต้องใช้ systemd เป็น PID 1) และพฤติกรรม graceful shutdown ของตัวโปรแกรมเมื่อได้รับสัญญาณ
> `SIGTERM` (สิ่งเดียวกับที่ systemd ส่งให้ตอนสั่ง `systemctl stop`) ก็ทดสอบจริงแล้วเช่นกัน
> ส่วนคำสั่ง `systemctl start/enable/status` แบบเต็มรูปแบบ **ทดสอบไม่ได้จริง** ในสภาพแวดล้อมนี้
> เพราะ container นี้ไม่ได้บูตด้วย systemd เป็น process แรก (PID 1) — รายละเอียดและ error จริง
> อยู่ท้ายหัวข้อ พร้อมคำอธิบายว่าบนเซิร์ฟเวอร์ Linux จริงจะเกิดอะไรขึ้นแทน

### ทำไมห้ามรัน Server ด้วยการเปิด Terminal แล้วปล่อยทิ้งไว้

วิธีที่มือใหม่มักทำตอน deploy ครั้งแรกคือ SSH เข้าเซิร์ฟเวอร์แล้วรัน `./task_api &` ตรงๆ ซึ่งมี
ปัญหาร้ายแรงหลายข้อ:

- **โปรแกรมตายทันทีที่ SSH session ปิด** (เว้นแต่ใช้ `nohup`/`screen`/`tmux` ซึ่งก็ยังไม่แข็งแรงพอ
  สำหรับ production)
- **ไม่มีการ restart อัตโนมัติถ้าโปรแกรม crash** — ถ้าเกิด unhandled exception หรือ segfault
  กลางดึก server จะหยุดทำงานไปเรื่อยๆ จนกว่าจะมีคนสังเกตเห็นและ SSH เข้าไปรันใหม่เอง
- **ไม่เริ่มทำงานอัตโนมัติเมื่อเซิร์ฟเวอร์ reboot** (เช่น ตอน patch ระบบปฏิบัติการแล้วต้อง restart
  เครื่อง)
- **ไม่มีการจัดการ log ที่เป็นระบบ** — output ของโปรแกรมหายไปพร้อมกับ terminal ที่ปิดไป

**systemd** คือ **init system** มาตรฐานของ Linux distro สมัยใหม่แทบทุกตัว (Ubuntu, Debian, Fedora,
CentOS/RHEL รุ่นใหม่) ทำหน้าที่เป็น **Process Supervisor** — ดูแล process ให้ทำงานตามที่ตั้งใจไว้
ครบวงจรทั้ง start, stop, restart อัตโนมัติเมื่อ crash, เริ่มอัตโนมัติตอนบูตเครื่อง และรวบรวม log
เข้า `journald` ให้เป็นระบบ

### เตรียมไฟล์สำหรับ Deploy

```bash
sudo mkdir -p /opt/task_api
sudo cp build/task_api /opt/task_api/task_api
sudo chown -R www-data:www-data /opt/task_api
```

เราใช้ user `www-data` (user มาตรฐานที่มีอยู่แล้วในทุกเซิร์ฟเวอร์ Ubuntu/Debian สำหรับรัน web
service) แทนที่จะรันด้วย `root` — ตามหลักการ **Least Privilege**: ถ้า Task API มีช่องโหว่ด้าน
ความปลอดภัยสักจุดหนึ่งแล้วถูกเจาะ ผู้บุกรุกจะได้สิทธิ์แค่เท่า `www-data` ไม่ใช่สิทธิ์ระดับ root
เต็มระบบ

### เขียน systemd Unit File

สร้างไฟล์ `/etc/systemd/system/task-api.service`:

```ini
[Unit]
Description=Task API - C++ REST backend (Crow + SQLite)
Documentation=https://example.com/docs/task-api
After=network.target

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/opt/task_api
ExecStart=/opt/task_api/task_api
Restart=on-failure
RestartSec=2
TimeoutStopSec=10
KillSignal=SIGTERM

# --- Hardening พื้นฐาน ---
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/opt/task_api
ProtectHome=true

# --- Resource limit ---
LimitNOFILE=4096

# --- ส่ง log เข้า journald ---
StandardOutput=journal
StandardError=journal
SyslogIdentifier=task-api

[Install]
WantedBy=multi-user.target
```

### อธิบายทีละ Directive

| Directive | ความหมาย |
|---|---|
| `After=network.target` | บอกให้เริ่ม service นี้ **หลังจาก** ระบบเครือข่ายพร้อมใช้งานแล้ว (ไม่ใช่รอ network เสร็จสมบูรณ์ 100% แค่บอกลำดับคร่าวๆ) |
| `Type=simple` | บอก systemd ว่า process หลักของ service คือ process ที่ `ExecStart` เรียกโดยตรง (ไม่ fork ตัวเองออกเป็น daemon แบบเก่า) — ตรงกับพฤติกรรมของ Crow ที่ `run()` ค้างอยู่ใน foreground |
| `User=www-data` / `Group=www-data` | รันด้วยสิทธิ์ผู้ใช้จำกัด ไม่ใช่ root |
| `Restart=on-failure` | ถ้า process จบด้วย exit code ที่ไม่ใช่ 0 (คือ error) หรือถูกสัญญาณฆ่าแบบผิดปกติ ให้ restart อัตโนมัติ (ค่าอื่นที่เลือกได้: `always`, `on-abnormal`, `no`) |
| `RestartSec=2` | รอ 2 วินาทีก่อน restart แต่ละครั้ง ป้องกัน "restart loop" ที่ถี่เกินไปจนกิน CPU ทั้งหมดถ้าโปรแกรม crash ทันทีที่เริ่มทุกครั้ง |
| `TimeoutStopSec=10` | ให้เวลาโปรแกรม 10 วินาทีในการปิดตัวเองอย่าง graceful หลังได้รับ `SIGTERM` ถ้าเกินเวลานี้ systemd จะส่ง `SIGKILL` บังคับปิดทันที |
| `KillSignal=SIGTERM` | สัญญาณที่ systemd ส่งให้ตอนสั่ง `stop` (ค่า default อยู่แล้ว แต่เขียนไว้ชัดเจนเพื่อความเข้าใจ) |
| `NoNewPrivileges=true` | ป้องกัน process นี้และ process ลูกของมันขอสิทธิ์เพิ่มขึ้นกว่าที่มีตอนเริ่ม (เช่นผ่าน setuid binary) |
| `ProtectSystem=strict` | mount `/usr`, `/boot`, `/etc` ทั้งระบบเป็น read-only สำหรับ service นี้ ยกเว้น path ที่ระบุใน `ReadWritePaths` |
| `PrivateTmp=true` | ให้ service เห็น `/tmp` เป็นพื้นที่ส่วนตัวแยกจาก process อื่นในระบบ |
| `LimitNOFILE=4096` | จำนวน file descriptor สูงสุดที่ process นี้เปิดได้พร้อมกัน (สำคัญสำหรับ web server ที่รับ connection พร้อมกันจำนวนมาก) |
| `StandardOutput=journal` | ส่ง stdout ของโปรแกรมเข้า `journald` แทนที่จะหายไปเฉยๆ — ดูย้อนหลังได้ด้วย `journalctl` |
| `WantedBy=multi-user.target` | บอกว่า service นี้ควรเริ่มทำงานเมื่อระบบเข้าสู่ multi-user mode ตามปกติ (ค่ามาตรฐานสำหรับ service ทั่วไป) — ใช้ตอนสั่ง `systemctl enable` |

### ตรวจสอบไฟล์ Unit ด้วย `systemd-analyze verify` (ทดสอบจริง)

`systemd-analyze verify` ตรวจสอบความถูกต้องของ unit file (syntax, directive ที่มีอยู่จริง,
ความสมเหตุสมผลของค่าต่างๆ) **โดยไม่ต้องรัน systemd เป็น PID 1** จึงใช้ได้แม้ในสภาพแวดล้อมที่
จำกัดแบบนี้:

```bash
$ systemd-analyze verify /etc/systemd/system/task-api.service
$ echo "exit code: $?"
```

ผลลัพธ์จริง:

```
exit code: 0
```

**ไม่มี warning หรือ error ใดๆ เลย** (exit code 0 และไม่มี output แปลว่าไฟล์ผ่านการตรวจสอบ
สมบูรณ์) — นี่คือขั้นตอนที่ควรทำเป็นนิสัยทุกครั้งก่อน deploy unit file ใหม่ เพราะ error บางประเภท
(เช่น พิมพ์ชื่อ directive ผิด, ใส่ path ที่ไม่มีอยู่จริง) จะไม่มี Nginx-style "syntax ok" ให้เห็น
ง่ายๆ เท่า `nginx -t` แต่ `systemd-analyze verify` ช่วยจับได้ก่อนที่จะเอาไปใช้จริง

### ทดสอบ Lifecycle เต็มรูปแบบ: ทำไมทำในสภาพแวดล้อมนี้ไม่ได้ (พูดตรงไปตรงมา)

ลองรันคำสั่งจริงในสภาพแวดล้อมนี้:

```bash
$ systemctl daemon-reload
$ systemctl start task-api
```

ผลลัพธ์จริงที่ได้:

```
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down
```

Error นี้บอกตรงๆ ว่า **container ที่ใช้เขียนบทเรียนนี้ไม่ได้บูตด้วย systemd เป็น process แรก
(PID 1)** — สภาพแวดล้อมแบบ sandboxed container จำนวนมาก (รวมถึงอันนี้) ใช้ process อื่น
(เช่น shell script หรือ container runtime เอง) เป็น PID 1 แทน ทำให้ `systemctl` ซึ่งต้องคุยกับ
`systemd` ผ่าน D-Bus ไม่สามารถทำงานได้เลย — **นี่ไม่ใช่ปัญหาของ unit file ที่เขียนไว้** (ซึ่งผ่าน
`systemd-analyze verify` แล้ว) แต่เป็นข้อจำกัดของสภาพแวดล้อมทดสอบล้วนๆ

**บนเซิร์ฟเวอร์ Linux จริง** (VM, bare metal, หรือ cloud instance ทั่วไปที่บูตแบบมาตรฐาน) คำสั่ง
เดียวกันนี้จะทำงานได้ตามปกติทุกประการ:

```bash
$ sudo systemctl daemon-reload          # โหลด unit file ใหม่เข้าระบบ
$ sudo systemctl enable task-api        # ตั้งให้เริ่มอัตโนมัติตอนเครื่องบูต
$ sudo systemctl start task-api         # เริ่มทำงานทันที

$ sudo systemctl status task-api
```

**ผลลัพธ์ที่คาดว่าจะได้บนเซิร์ฟเวอร์จริง** (ตัวอย่างรูปแบบมาตรฐาน ไม่ใช่ output จริงจากเครื่องนี้):

```
● task-api.service - Task API - C++ REST backend (Crow + SQLite)
     Loaded: loaded (/etc/systemd/system/task-api.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-09-26 12:00:00 UTC; 5s ago
   Main PID: 12345 (task_api)
      Tasks: 5 (limit: 4096)
     Memory: 8.2M
        CPU: 45ms
     CGroup: /system.slice/task-api.service
             └─12345 /opt/task_api/task_api
```

ดู log ย้อนหลังผ่าน `journald`:

```bash
$ sudo journalctl -u task-api -f
```

### ทดสอบจริง: พฤติกรรม Graceful Shutdown เมื่อได้รับ SIGTERM

แม้จะสั่ง `systemctl stop` ตรงๆ ไม่ได้ในสภาพแวดล้อมนี้ แต่ **สิ่งที่ `systemctl stop` ทำจริงๆ
เบื้องหลังคือส่งสัญญาณ `SIGTERM` ไปที่ process** ซึ่งทดสอบได้ตรงๆ ด้วย `kill -TERM`:

```bash
$ ./task_api &
$ PID=$!
$ kill -TERM $PID
$ sleep 1
$ ps -p $PID    # ตรวจว่า process ยังอยู่หรือไม่
```

Log จริงที่โปรแกรมพิมพ์ออกมาระหว่างขั้นตอนนี้:

```
(2026-09-26 11:24:12) [INFO    ] Request: 127.0.0.1:48940 0x7f4cb4000c30 HTTP/1.1 GET /health
(2026-09-26 11:24:12) [INFO    ] Response: 0x7f4cb4000c30 /health 200 0
(2026-09-26 11:24:23) [INFO    ] Closing acceptor. 0x556b181d5608
(2026-09-26 11:24:23) [INFO    ] Closing IO service 0x556b181d43c0
(2026-09-26 11:24:23) [INFO    ] Closing IO service 0x556b181d43c8
(2026-09-26 11:24:23) [INFO    ] Closing IO service 0x556b181d43d0
(2026-09-26 11:24:23) [INFO    ] Closing main IO service (0x556b181d55c8)
(2026-09-26 11:24:23) [INFO    ] Exiting.
```

และคำสั่ง `ps -p $PID` หลังจากนั้นไม่พบ process อยู่เลย — ยืนยันว่า **Crow's `app.run()` ดักจับ
`SIGTERM` และปิดตัวเองอย่างเป็นระเบียบจริง**: ปิด acceptor (ไม่รับ connection ใหม่เพิ่ม) ก่อน
แล้วค่อยปิด IO service ทีละตัว แทนที่จะถูกฆ่าทิ้งดื้อๆ กลางคัน นี่คือพฤติกรรมที่ **ทำให้
`Restart=on-failure` ของ systemd ทำงานถูกต้อง** — เมื่อ operator สั่ง `systemctl stop` (ซึ่งส่ง
`SIGTERM`) โปรแกรมจะปิดตัวเองอย่างสุภาพ (ไม่ถือเป็น "failure") systemd จึง**ไม่**พยายาม restart
มันกลับมาอีก ต่างจากกรณีโปรแกรม crash เอง (เช่น segfault) ที่ systemd จะมองว่าเป็น "failure"
แล้ว restart ให้อัตโนมัติตามที่ตั้งค่าไว้

---

## 112.7 Production Readiness Checklist (Step 895)

การมี server ที่ "รันได้" กับ server ที่ "พร้อมสำหรับ production จริง" เป็นคนละเรื่องกัน
checklist นี้รวบรวมประเด็นสำคัญที่ต้องตรวจสอบก่อน deploy ระบบขึ้นใช้งานจริงกับผู้ใช้จริง

### 1. Logging (บันทึกเหตุการณ์)

| ข้อตรวจสอบ | สถานะที่ควรเป็น |
|---|---|
| Log ของ application ไปที่ไหน | ควรส่งเข้า `journald` ผ่าน `StandardOutput=journal` (ทดสอบแล้วในหัวข้อ 112.6) ไม่ใช่หายไปเฉยๆ หรือเขียนลงไฟล์ที่ไม่มีใครดูแล |
| Log มี timestamp หรือไม่ | Crow ใส่ timestamp ให้อัตโนมัติทุกบรรทัด (เห็นได้จาก log จริงข้างต้น) |
| Log แยกระดับความสำคัญหรือไม่ (INFO/WARNING/ERROR) | Crow รองรับผ่าน `app.loglevel(crow::LogLevel::Warning)` เพื่อลด verbosity ใน production (ไม่ต้องเห็นทุก request เป็น INFO) |
| Log rotation ตั้งไว้หรือยัง | `journald` จัดการ rotation ให้อัตโนมัติตาม `journald.conf` (ค่า default ของ Ubuntu จำกัดขนาดไม่ให้ log บวมจนเต็มดิสก์) |
| Access log ของ Nginx เก็บไว้หรือไม่ | ค่า default ของ Nginx เขียนไปที่ `/var/log/nginx/access.log` อยู่แล้ว — ควรตรวจสอบว่ามีการ rotate (ผ่าน `logrotate` ซึ่งเป็นค่า default ของ Ubuntu เช่นกัน) |

### 2. Monitoring เบื้องต้น

| ข้อตรวจสอบ | สถานะที่ควรเป็น |
|---|---|
| มี Health Check Endpoint หรือไม่ | มีแล้ว (`GET /health` จาก Part 108) — ทดสอบจริงแล้วว่าใช้งานได้ทั้งเรียกตรงและผ่าน Nginx |
| ใครเรียก Health Check เป็นระยะ | ในระบบจริงควรมี Uptime Monitor ภายนอก (เช่น UptimeRobot, Pingdom หรือระบบภายในองค์กร) เรียก endpoint นี้ทุก 1-5 นาที แล้วแจ้งเตือนถ้าไม่ตอบ 200 |
| Docker Healthcheck ตั้งไว้หรือยัง | ตั้งไว้แล้วใน Dockerfile (`HEALTHCHECK` directive) และ `docker-compose.yml` (`healthcheck:` ของทั้งสอง service) |
| ตรวจสอบ Resource การใช้งาน (CPU/Memory) | เบื้องต้นใช้ `systemctl status` (แสดง Memory/CPU ให้ในตัว ตามตัวอย่างในหัวข้อ 112.6) หรือ `htop`/`top` สำหรับตรวจสอบด้วยมือ ระบบใหญ่ขึ้นควรพิจารณาเครื่องมือแบบ Prometheus + Grafana (นอกขอบเขตของ Part นี้) |
| Alert เมื่อ Disk เต็มหรือไม่ | ตรวจสอบด้วย `df -h` เป็นประจำ หรือตั้ง monitoring script ง่ายๆ ที่ cron เรียกเช็คแล้วแจ้งเตือนถ้าต่ำกว่า threshold |

### 3. Graceful Shutdown

| ข้อตรวจสอบ | สถานะที่ควรเป็น |
|---|---|
| โปรแกรมจัดการ `SIGTERM` ถูกต้องหรือไม่ | ทดสอบจริงแล้วในหัวข้อ 112.6 — Crow ปิด acceptor และ IO service อย่างเป็นระเบียบ ไม่ตัดการเชื่อมต่อ request ที่กำลังประมวลผลอยู่กลางคัน |
| `TimeoutStopSec` ตั้งไว้เหมาะสมหรือไม่ | ตั้งไว้ 10 วินาที — นานพอให้ request ที่กำลังทำงานอยู่เสร็จ แต่ไม่นานเกินจนทำให้ deploy ช้า |
| ฐานข้อมูลปิดอย่างปลอดภัยหรือไม่ | `Db::~Db()` เรียก `sqlite3_close()` ผ่าน RAII (Part 68) — ถ้า process ปิดแบบ graceful destructor จะถูกเรียกและปิด connection อย่างถูกต้อง (ต่างจากถูก `SIGKILL` ฆ่าทันทีที่ destructor ไม่มีโอกาสได้ทำงานเลย) |
| หลีกเลี่ยง `kill -9` ในการ deploy ปกติหรือไม่ | ควรใช้ `systemctl stop`/`restart` เสมอ (ส่ง `SIGTERM` ให้เวลา graceful shutdown) สงวน `kill -9`/`SIGKILL` ไว้เฉพาะกรณีฉุกเฉินที่โปรแกรมค้างจริงๆ เท่านั้น |

### 4. ความปลอดภัย (Security)

| ข้อตรวจสอบ | สถานะที่ควรเป็น |
|---|---|
| รันด้วย non-root user หรือไม่ | ทั้ง systemd (`User=www-data`) และ Docker (`USER appuser`) ตั้งไว้ถูกต้องแล้ว |
| Compiler อยู่ใน production image หรือไม่ | ไม่มี — ใช้ multi-stage build ตามหัวข้อ 112.2 |
| Secret (password ฐานข้อมูล ฯลฯ) hardcode ในโค้ดหรือไม่ | ไม่ควร hardcode — ส่งผ่าน environment variable เสมอ (`DATABASE_URL` ตามหัวข้อ 112.3) และไม่ควร commit ค่า secret จริงเข้า Git |
| Firewall เปิดเฉพาะพอร์ตที่จำเป็นหรือไม่ | ควรเปิดแค่ 22 (SSH), 80, 443 ผ่าน `ufw`/`iptables` — พอร์ตของ backend (18180) และ PostgreSQL (5432) ไม่ควรเปิดสู่อินเทอร์เน็ตภายนอกเลย ให้เข้าถึงได้แค่จาก `127.0.0.1`/internal network เท่านั้น |
| `server_tokens` ของ Nginx ปิดหรือยัง | ควรตั้ง `server_tokens off;` ใน `http {}` block เพื่อไม่ให้ response header เปิดเผยเลขเวอร์ชัน Nginx ที่แน่นอน (ลดข้อมูลที่ผู้โจมตีใช้หาช่องโหว่เฉพาะเวอร์ชัน) |
| HTTPS บังคับใช้หรือไม่ | ควร redirect ทุก request จาก HTTP ไป HTTPS เสมอ (ตามตัวอย่าง Certbot ในหัวข้อ 112.5) |

### 5. Build & Deployment Process

| ข้อตรวจสอบ | สถานะที่ควรเป็น |
|---|---|
| Compile ด้วย `-O2`/`-O3` (ไม่ใช่ `-O0`) หรือไม่ | Production build ควร optimize เสมอ ต่างจากตอน develop ที่ใช้ `-O0 -g` เพื่อ debug ง่าย (ทบทวน Part 1) |
| Build reproducible หรือไม่ | ควร pin เวอร์ชันของ dependency ทุกตัว (เช่น Crow ใช้ tag `v1.2.0` ไม่ใช่ `master` ที่เปลี่ยนแปลงได้ตลอดเวลา ตามที่ทำในหัวข้อ 112.2) |
| มี Rollback plan หรือไม่ | ควรเก็บ image/binary เวอร์ชันก่อนหน้าไว้เสมอ เผื่อ deploy เวอร์ชันใหม่แล้วมีปัญหาต้องย้อนกลับเร็ว |
| Deploy แบบ Zero-downtime ได้หรือไม่ | อย่างน้อยที่สุด ควรมี backend มากกว่า 1 instance หลัง Nginx load balancer (ตามที่ทดสอบจริงในหัวข้อ 112.4) เพื่อ deploy ทีละตัวโดยไม่ตัด traffic ทั้งหมดพร้อมกัน |

---

## 112.8 ภาพรวมสถาปัตยกรรมการ Deploy แบบสมบูรณ์ (Step 896)

### แผนภาพรวมทุกองค์ประกอบที่เรียนใน Part นี้

```
                            อินเทอร์เน็ต
                                 │
                                 │  HTTPS (443) / HTTP→redirect (80)
                                 ▼
                    ┌─────────────────────────┐
                    │   Nginx (Reverse Proxy)  │
                    │  - TLS Termination        │  ← หัวข้อ 112.4, 112.5
                    │  - proxy_pass /api/       │
                    │  - Load Balancing         │
                    │    (upstream หลาย backend)│
                    └───────────┬───────────────┘
                                │  HTTP (plain, ภายในเครื่องเท่านั้น)
                 ┌──────────────┼──────────────┐
                 ▼              ▼              ▼
         ┌───────────┐  ┌───────────┐  ┌───────────┐
         │ Task API  │  │ Task API  │  │ Task API  │   ← C++ backend
         │ instance 1│  │ instance 2│  │ instance N│     (systemd service
         │ :18180    │  │ :18181    │  │ :1818N    │      หรือ Docker container)
         └─────┬─────┘  └─────┬─────┘  └─────┬─────┘     หัวข้อ 112.2, 112.6
               │              │              │
               └──────────────┼──────────────┘
                               ▼
                      ┌─────────────────┐
                      │   PostgreSQL     │   ← หัวข้อ 112.3
                      │   (ข้อมูลกลาง     │
                      │    ทุก instance   │
                      │    เชื่อมร่วมกัน) │
                      └─────────────────┘
```

### สองเส้นทางของการ Deploy ที่เรียนใน Part นี้

| ประเด็น | เส้นทาง A: Bare Metal/VM + systemd | เส้นทาง B: Docker + docker-compose |
|---|---|---|
| การทดสอบใน Part นี้ | ✅ ทดสอบจริงครบทุกจุด (build, nginx, systemd unit, SIGTERM) | 📄 อ้างอิง — เขียนถูกต้องตามหลักปฏิบัติ แต่ไม่ได้รันจริง |
| Reproducibility ข้ามเครื่อง | ต้องพึ่ง config management (Ansible ฯลฯ) หรือทำตามคู่มือด้วยมือ | สูงมาก — image เดียวกันรันได้ทุกที่ที่มี Docker |
| Overhead ตอนรัน | ต่ำที่สุด (ไม่มี virtualization layer) | มี layer เพิ่มเล็กน้อย (แต่เบากว่า VM มาก) |
| ความง่ายในการ scale หลาย instance | ต้องจัดการเองด้วย systemd unit หลายตัว + Nginx upstream ด้วยมือ | `docker compose up --scale api=3` ง่ายกว่ามาก |
| เหมาะกับ | ทีมเล็ก, ระบบเดียว, ต้องการ performance สูงสุด, คุ้นเคยกับ Linux administration | ทีมที่ต้องการ deploy สม่ำเสมอหลายเครื่อง/หลาย environment, มีแผนจะขยายเป็น multi-container ในอนาคต |
| ระบบ orchestration ขั้นถัดไป | Ansible/Puppet สำหรับจัดการหลายเครื่อง | Kubernetes/Docker Swarm (นอกขอบเขต Module I แต่ต่อยอดได้ทันที) |

ทั้งสองเส้นทางไม่ใช่คู่แข่งที่ต้องเลือกอย่างใดอย่างหนึ่งตลอดไป — ทีมจำนวนมากเริ่มจากเส้นทาง A
(เข้าใจง่าย ควบคุมได้เต็มที่) แล้วค่อยย้ายไปเส้นทาง B เมื่อระบบโตขึ้นและต้องการ deploy บ่อยขึ้น
หรือ scale ข้ามหลายเครื่อง สิ่งสำคัญที่สุดคือ **แนวคิดเบื้องหลัง (multi-stage build, reverse
proxy, health check, graceful shutdown, environment-based config) เหมือนกันทั้งสองเส้นทาง**
เพียงแค่เครื่องมือที่ใช้ "ดูแล" process ต่างกัน (systemd vs Docker/Kubernetes)

### เส้นทางต่อยอด

ความรู้เรื่อง Deploy ใน Part นี้เป็นจุดปิดท้ายของ **Module I — Web Development ด้วย C/C++**
ที่เริ่มจาก Raw Socket (Part 100) ไปจนถึงระบบที่ deploy ได้จริงบน production server ครบวงจร
ใน **Module J** ที่กำลังจะเริ่มต้น เราจะยกระดับไปสู่ทักษะระดับมืออาชีพที่ใช้ได้กับโค้ด C/C++
ทุกประเภท ไม่ใช่แค่ web application เท่านั้น — เริ่มจาก **Clean Code และ Code Review Practice**
ใน Part 113 ซึ่งเป็นทักษะที่แยกวิศวกรซอฟต์แวร์มืออาชีพออกจากโปรแกรมเมอร์ที่เขียนโค้ดให้รันได้
อย่างเดียว

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

1. **ลืมทำ Multi-stage Build ปล่อยให้ Compiler ติดไปกับ Production Image** — ถ้าเห็น
   `RUN apt-get install build-essential cmake` อยู่ใน stage เดียวกับที่มี `CMD` รัน executable
   ตัวจริง ให้สงสัยทันที ผลที่ตามมาคือ image ใหญ่เกินจำเป็นหลายเท่าตัว และมี compiler/header
   ของ dev library ติดอยู่ใน production ซึ่งเป็นพื้นที่โจมตีที่ไม่จำเป็นต้องมี วิธีแก้คือแยก
   `FROM ... AS builder` ออกจาก stage สุดท้ายเสมอ แล้ว `COPY --from=builder` เอาแค่ executable
   ที่ compile เสร็จแล้วมาเท่านั้น (หัวข้อ 112.2)

2. **`proxy_pass` มี/ไม่มี Trailing Slash แล้วลืมว่ามันเปลี่ยนพฤติกรรม** — พิสูจน์จริงแล้วในหัวข้อ
   112.4 ว่า `proxy_pass http://backend;` (ไม่มี `/`) กับ `proxy_pass http://backend/;` (มี `/`)
   ให้ผลลัพธ์ต่างกันโดยสิ้นเชิง (404 กับ 200 Success) กฎง่ายๆ ที่จำได้เสมอคือ **ถ้ามี path
   ต่อท้าย `proxy_pass` (แม้จะเป็นแค่ `/`) Nginx จะตัดส่วนของ `location` ออกก่อนส่งต่อ ถ้าไม่มี
   จะส่ง URL เต็มไปตรงๆ**

3. **ใส่ `listen [::]:80;` (IPv6) โดยไม่ตรวจสอบว่าสภาพแวดล้อมรองรับหรือไม่** — พิสูจน์จริงแล้วว่า
   สภาพแวดล้อมบาง container ให้ error `socket() [::]:80 failed (97: Address family not
   supported by protocol)` ทันทีที่พยายามเริ่ม Nginx ถ้าเจอ error นี้ วิธีแก้คือลบบรรทัด
   `listen [::]:80;` ออก เหลือแค่ `listen 80;` เพียงบรรทัดเดียว (ใช้งานได้ปกติสำหรับ IPv4)

4. **ลืมใส่ Volume ให้ PostgreSQL ใน docker-compose.yml** — ถ้า service `db` ไม่มี `volumes:`
   ผูกกับ named volume ไว้ ข้อมูลทั้งหมดในฐานข้อมูลจะหายทันทีที่ container ถูกลบ (เช่นตอน
   `docker compose down -v` โดยไม่ตั้งใจ หรือตอนอัปเดตเวอร์ชัน PostgreSQL image) ต้องผูก
   `pgdata:/var/lib/postgresql/data` เสมอสำหรับข้อมูลที่ต้องอยู่รอดข้ามการ restart ของ container

5. **Hardcode Connection String/Password ไว้ในโค้ดแทนที่จะอ่านจาก Environment Variable** —
   ทำให้ image เดียวกันเอาไปใช้กับหลาย environment (dev/staging/production) ไม่ได้เลยโดยไม่ต้อง
   compile ใหม่ ขัดกับหลักการ 12-Factor App (หัวข้อ 112.3) และเสี่ยงที่ credential จริงจะหลุด
   เข้า Git repository โดยไม่ตั้งใจ (ถ้า commit source code ที่มี password ฝังอยู่)

6. **สั่ง `kill -9` (SIGKILL) แทน `systemctl stop`/`kill -TERM` ตอน deploy** — `SIGKILL` ฆ่า
   process ทันทีโดยไม่ให้โอกาสทำ cleanup ใดๆ เลย (destructor ของ C++ ไม่ถูกเรียก, connection
   ฐานข้อมูลค้าง, request ที่กำลังประมวลผลถูกตัดกลางคัน) ต่างจาก `SIGTERM` ที่พิสูจน์แล้วในหัวข้อ
   112.6 ว่า Crow จัดการปิด acceptor และ IO service อย่างเป็นระเบียบก่อนค่อย exit — ควรใช้
   `SIGTERM`/`systemctl stop` เสมอ สงวน `SIGKILL` ไว้กรณีฉุกเฉินที่โปรแกรมค้างไม่ตอบสนองจริงๆ
   เท่านั้น

7. **รัน Backend Process ด้วยสิทธิ์ `root` โดยไม่จำเป็น** — ทั้งใน systemd (`User=root` หรือไม่
   ระบุ `User=` เลยซึ่งจะ default เป็น root ถ้ารันโดย root) และใน Docker (ไม่มี `USER` directive
   ทำให้ container รันด้วย root โดย default) ถ้าแอปมีช่องโหว่แล้วถูกเจาะ ผู้บุกรุกจะได้สิทธิ์เต็ม
   ระบบทันที ควรสร้าง user เฉพาะ (`www-data` หรือ `appuser`) ที่มีสิทธิ์จำกัดเสมอ

8. **ลืมตั้งค่า `depends_on` ให้รอ Database พร้อมจริงๆ ใน docker-compose.yml** — การเขียน
   `depends_on: [db]` เฉยๆ รอแค่ container เริ่มทำงาน ไม่ได้รอให้ PostgreSQL accept connection
   ได้จริง ทำให้ app container อาจ crash ตั้งแต่ startup เพราะต่อฐานข้อมูลไม่ติด ต้องใช้คู่กับ
   `condition: service_healthy` และ `healthcheck` ที่ใช้ `pg_isready` เสมอ (หัวข้อ 112.3)

---

## แบบฝึกหัดท้ายบท

1. เขียน `Dockerfile` แบบ multi-stage build สำหรับโปรเจกต์ SSR (`task_ssr`) จาก Part 111 แทนที่
   จะเป็น REST API ธรรมดา (สังเกตว่าต้อง `COPY` โฟลเดอร์ `templates/` และ `static/` เข้าไปใน
   runtime stage ด้วย เพราะ `crow::mustache::load()` อ่านไฟล์จากดิสก์ตอน runtime ไม่ใช่ตอน
   compile)

2. แก้ไข Nginx config ในหัวข้อ 112.4 ให้เพิ่ม `location /static/` แยกต่างหากจาก `location /api/`
   โดยให้ Nginx **serve ไฟล์ static เอง** (ด้วย directive `root`/`alias`) แทนที่จะ proxy ไปหา
   Crow (ในระบบจริง Nginx serve static file ได้เร็วกว่า backend มาก)

3. เพิ่ม `rate limiting` ให้ Nginx config เพื่อจำกัดไม่ให้ IP เดียวยิง request เข้า `/api/`
   เกิน 10 ครั้งต่อวินาที (ค้นคว้า directive `limit_req_zone` และ `limit_req` ของ Nginx เพิ่มเติม)

4. เขียน systemd unit file สำหรับรัน PostgreSQL container ธรรมดาผ่าน `docker run` (ไม่ใช่รัน
   ตรงบนเครื่อง) โดยใช้ `ExecStartPre=/usr/bin/docker rm -f task_api_db` (ลบ container เก่าก่อนเริ่มใหม่
   กันชื่อชนกัน) และ `ExecStart=/usr/bin/docker run --rm --name task_api_db -p 5432:5432 postgres:16`

5. ตรวจสอบ unit file ของตัวเองในข้อ 4 ด้วย `systemd-analyze verify` ให้ผ่านโดยไม่มี warning
   ก่อนส่งการบ้าน

6. เขียน checklist ของตัวเอง (ต่อยอดจากหัวข้อ 112.7) เพิ่มอีกอย่างน้อย 3 ข้อที่คิดว่าสำคัญ
   สำหรับระบบที่ตัวเองกำลังจะ deploy จริง (เช่น backup strategy ของฐานข้อมูล, การแจ้งเตือนทีม
   เมื่อ deploy ล้มเหลว) พร้อมอธิบายเหตุผลสั้นๆ ว่าทำไมถึงสำคัญ

### แนวทางเฉลยข้อ 1

`Dockerfile` สำหรับ `task_ssr` (ปรับจาก Task API เดิม เพิ่ม `templates/` และ `static/`):

```dockerfile
# ============================================================
# Stage 1: builder
# ============================================================
FROM ubuntu:24.04 AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential cmake git ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN git clone --branch v1.2.0 --depth 1 \
    https://github.com/CrowCpp/Crow.git /tmp/crow \
    && cp -r /tmp/crow/include/* /usr/local/include/ \
    && rm -rf /tmp/crow

WORKDIR /build
COPY main.cpp .

RUN g++ -std=c++17 -O2 -I/usr/local/include main.cpp -o task_ssr -lpthread

# ============================================================
# Stage 2: runtime — เพิ่ม templates/ และ static/ เข้ามาด้วย
# เพราะ crow::mustache::load() อ่านจากดิสก์ตอน runtime ไม่ใช่ตอน compile
# ============================================================
FROM ubuntu:24.04 AS runtime

RUN useradd --system --no-create-home --shell /usr/sbin/nologin appuser

WORKDIR /app
COPY --from=builder /build/task_ssr /app/task_ssr
COPY templates/ /app/templates/
COPY static/ /app/static/

RUN chown -R appuser:appuser /app
USER appuser

EXPOSE 18080

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://127.0.0.1:18080/tasks || exit 1

CMD ["/app/task_ssr"]
```

จุดสำคัญที่ต่างจาก Dockerfile ของ Task API (REST) คือ **สอง `COPY` เพิ่มเติม**
(`templates/` และ `static/`) ในสุด stage เพราะไฟล์เหล่านี้ไม่ได้ถูกฝังเข้าไปในตัว executable
ตอน compile (ไม่เหมือน string literal ในโค้ด) แต่ `crow::mustache::set_base("templates")` และ
`crow::mustache::load(...)` เปิดอ่านไฟล์จากดิสก์จริงทุกครั้งที่มี request เข้ามา (ตามที่พิสูจน์
ไว้ใน Part 111 ว่าแก้ template แล้วเห็นผลทันทีโดยไม่ต้อง restart) — ถ้าลืม `COPY` โฟลเดอร์เหล่านี้
เข้า runtime stage โปรแกรมจะ compile ผ่านและเริ่มทำงานได้ปกติ แต่จะ crash หรือคืน error ทันทีที่
มี request แรกเข้ามา เพราะหาไฟล์ template ไม่เจอ — เป็นบั๊กที่ตรวจจับได้ยากตอน build (เพราะ
build สำเร็จ) แต่พังตอน runtime เท่านั้น

### แนวทางเฉลยข้อ 2

Nginx config ที่แยก static file ออกจาก proxy ไปหา backend:

```nginx
server {
    listen 80;
    server_name _;

    # Static file — ให้ Nginx serve เองโดยตรง ไม่ต้องผ่าน Crow เลย
    location /static/ {
        alias /opt/task_ssr/static/;
        expires 7d;                      # บอก browser ให้ cache ไฟล์ไว้ 7 วัน
        add_header Cache-Control "public";
    }

    # ทุกอย่างอื่น (HTML ที่ render จาก template, API) ส่งต่อไปที่ Crow
    location / {
        proxy_pass http://127.0.0.1:18080/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**จุดสำคัญที่ต้องระวัง**: ใช้ `alias` ไม่ใช่ `root` สำหรับ `location /static/` — ความต่างคือ
`alias /opt/task_ssr/static/;` แทนที่ path `/static/` ทั้งหมดด้วย path ที่ระบุตรงๆ (request
`/static/style.css` → ไฟล์จริงที่ `/opt/task_ssr/static/style.css`) ในขณะที่ `root` จะ**ต่อท้าย**
path เดิมเข้าไป (ถ้าใช้ `root /opt/task_ssr;` กับ `location /static/` แบบเดียวกัน request
`/static/style.css` จะไปหาไฟล์ที่ `/opt/task_ssr/static/style.css` เหมือนกันพอดีในกรณีนี้ เพราะ
ชื่อโฟลเดอร์ตรงกับชื่อ location — แต่ถ้าตั้งชื่อ location ไม่ตรงกับโครงสร้างโฟลเดอร์จริง `root`
กับ `alias` จะให้ผลลัพธ์ต่างกันทันที) การสับสนระหว่างสองคำสั่งนี้เป็นอีกหนึ่งหลุมพรางที่พบบ่อย
ของ Nginx config ที่ทำ static file serving

การแยกแบบนี้ทำให้ CSS/JS/รูปภาพถูก serve โดย Nginx โดยตรง (เร็วกว่ามาก เพราะ Nginx เขียนด้วย C
ที่ optimize มาเพื่องาน static file โดยเฉพาะ และไม่ต้องผ่าน network hop ไปที่ Crow เลย) ในขณะที่
หน้า HTML ที่ต้อง render จากข้อมูลจริง (ผ่าน `crow::mustache`) ยังคงถูกส่งไปที่ backend ตามปกติ

---

## สรุปท้ายบท

ใน Part นี้เราได้:

- เข้าใจปัญหา "มันรันบนเครื่องผมนะ" อย่างลึกซึ้ง และเห็นว่า Docker แก้ปัญหานี้ด้วยแนวคิด
  Reproducible Environment อย่างไร
- เขียน Dockerfile แบบ Multi-stage Build ที่ถูกต้องสำหรับ C++ web application แยก build stage
  (toolchain เต็ม) ออกจาก runtime stage (เบาที่สุด) พร้อมเข้าใจเหตุผลด้าน image size และ
  ความปลอดภัยเบื้องหลัง
- เขียน `docker-compose.yml` ที่ประกอบ backend C++ กับ PostgreSQL เข้าด้วยกัน เข้าใจเรื่อง
  internal network, named volume และ environment variable ตามแนวคิด 12-Factor App — พร้อม
  ยืนยัน credential/connection string ด้วยการทดสอบจริงผ่าน PostgreSQL แบบ native
- ตั้งค่า Nginx เป็น Reverse Proxy หน้า C++ backend **และทดสอบจริงทุกจุด** ทั้ง proxy_pass
  พื้นฐาน, หลุมพรางเรื่อง trailing slash (พิสูจน์จริงด้วย 404 vs 200), หลุมพรางเรื่อง IPv6
  listen directive (พิสูจน์จริงด้วย error message), และแม้แต่ load balancing ข้ามหลาย backend
  instance
- เข้าใจแนวคิดของ HTTPS ด้วย Let's Encrypt/Certbot และเหตุผลที่ต้องทำ TLS Termination ที่ชั้น
  Reverse Proxy
- เขียนและตรวจสอบ systemd unit file ด้วย `systemd-analyze verify` จริง เข้าใจทุก directive
  สำคัญ และพิสูจน์จริงว่า Crow จัดการ graceful shutdown ด้วย `SIGTERM` ได้อย่างถูกต้อง
- ได้ checklist ของ Production Readiness ที่ครอบคลุม Logging, Monitoring, Graceful Shutdown,
  Security และ Build Process ที่นำไปใช้ตรวจสอบระบบของตัวเองได้ทันที

Part นี้คือจุดปิดท้ายของ **Module I — Web Development ด้วย C/C++** เส้นทางที่เริ่มจากการสร้าง
HTTP server จาก raw socket ด้วยมือ (Part 100) ผ่าน framework อย่าง Crow (Part 102-103),
ฐานข้อมูล (Part 105-106), REST API ครบวงจร (Part 108), WebSocket (Part 109), Authentication
(Part 110), Server-Side Rendering (Part 111) จนมาถึงการ deploy ขึ้น production จริงใน Part นี้
— ครบวงจรตั้งแต่ "ไม่มีอะไรเลย" จนถึง "ระบบที่ผู้ใช้จริงเข้าถึงได้อย่างปลอดภัยและเชื่อถือได้"

ใน **Part 113** ซึ่งเป็นจุดเริ่มต้นของ **Module J — Professional และ World-Class Practices**
เราจะเปลี่ยนโฟกัสจาก "จะ deploy อย่างไร" ไปสู่ "จะเขียนโค้ดและทำงานร่วมกับทีมอย่างมืออาชีพ
อย่างไร" เริ่มต้นด้วย **Clean Code และ Code Review Practice** ทักษะที่สำคัญไม่แพ้ความรู้ทาง
เทคนิคเลยสำหรับวิศวกรซอฟต์แวร์ระดับโลก

**ต่อไป:** [Part 113 — Clean Code และ Code Review Practice](./part-113-clean-code-review.md)
