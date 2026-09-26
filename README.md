# สอนเขียนและพัฒนาโปรแกรมและเว็บแอปพลิเคชันด้วย C / C++

หลักสูตรภาษาไทยแบบ Step-by-Step (Step 1–1000) สอนตั้งแต่ระดับพื้นฐานที่สุด
จนถึงระดับมืออาชีพ/ระดับโลก ครอบคลุมทั้งภาษา C, C++ สมัยใหม่ (Modern C++),
Systems Programming, Data Structures & Algorithms, และการพัฒนาเว็บแอปพลิเคชัน
ด้วย C/C++ (REST API, Database, WebSocket, Docker Deployment)

- ดูโครงสร้างหลักสูตรทั้งหมด (125 Part / 11 Module): [`CURRICULUM.md`](./CURRICULUM.md)
- เนื้อหาแต่ละ Part อยู่ในโฟลเดอร์: [`parts/`](./parts)
- ตัวอย่างโค้ด/โปรเจกต์เสริมอยู่ในโฟลเดอร์: [`code/`](./code)

## วิธีใช้หลักสูตรนี้

1. เรียงตาม Part จากน้อยไปมาก (Part 1 → Part 125) อย่าข้าม เพราะแต่ละ Part
   ต่อยอดจาก Part ก่อนหน้า
2. ลงมือพิมพ์โค้ดตัวอย่างเองทุกครั้ง อย่า copy-paste เฉยๆ แล้วรันดูผลลัพธ์จริง
3. ทำแบบฝึกหัดท้ายบททุก Part ก่อนไป Part ถัดไป
4. เครื่องมือที่ต้องมี: GCC/Clang, CMake, Git, และ Text Editor/IDE (แนะนำ VS Code)

## สถานะความคืบหน้าของเนื้อหา

ตารางด้านล่างอัปเดต ณ ปัจจุบัน (✅ = เขียนเสร็จแล้ว, ⏳ = กำลังเขียน/ยังไม่เขียน)

### Module A — รากฐานภาษา C (Part 1–10)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 01 | เริ่มต้นกับภาษา C | ✅ |
| 02 | ตัวแปร ชนิดข้อมูล และตัวดำเนินการ | ✅ |
| 03 | การรับส่งข้อมูลเชิงลึก (printf/scanf) | ✅ |
| 04 | โครงสร้างควบคุมเงื่อนไข | ✅ |
| 05 | การวนซ้ำ (Loop) | ✅ |
| 06 | ฟังก์ชันในภาษา C | ✅ |
| 07 | Array และ String ใน C | ✅ |
| 08 | Pointer พื้นฐาน (ตอนที่ 1) | ✅ |
| 09 | Pointer ขั้นสูง (ตอนที่ 2) | ✅ |
| 10 | Struct, Union, Enum, typedef | ✅ |

### Module B — C ระดับกลางและ DS&A ด้วย C (Part 11–25)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 11 | Dynamic Memory Allocation | ✅ |
| 12 | Array หลายมิติและ Pointer-to-Pointer | ✅ |
| 13 | File I/O ใน C | ✅ |
| 14 | Preprocessor และ Macro | ✅ |
| 15 | Bit Manipulation | ✅ |
| 16 | Error Handling ใน C | ✅ |
| 17 | Modular Programming (Header/Linkage) | ✅ |
| 18 | Makefile และ Build Automation | ✅ |
| 19 | Linked List (Singly/Doubly/Circular) | ✅ |
| 20 | Stack และ Queue | ✅ |
| 21 | Tree เบื้องต้น (Binary Tree, BST) | ✅ |
| 22 | Hash Table ด้วย C | ✅ |
| 23 | Sorting Algorithm | ✅ |
| 24 | Searching Algorithm และ Big-O | ✅ |
| 25 | Recursion ขั้นสูงและ Dynamic Programming | ✅ |

### Module C — Systems Programming (Part 26–40)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 26 | พื้นฐานระบบปฏิบัติการสำหรับโปรแกรมเมอร์ | ✅ |
| 27 | Process Management (fork/exec/wait) | ✅ |
| 28 | Signal และ Signal Handling | ✅ |
| 29 | Inter-Process Communication (Pipe/FIFO) | ✅ |
| 30 | Shared Memory และ Message Queue | ✅ |
| 31 | POSIX Threads เบื้องต้น | ✅ |
| 32 | Thread Synchronization | ✅ |
| 33 | Socket Programming เบื้องต้น (TCP) | ✅ |
| 34 | Socket ขั้นสูง (UDP/select/poll/epoll) | ✅ |
| 35 | โปรเจกต์ TCP Chat Server/Client | ✅ |
| 36 | Memory-Mapped File (mmap) | ✅ |
| 37 | การ Debug ด้วย GDB | ✅ |
| 38 | Valgrind และ Memory Leak Detection | ✅ |
| 39 | Static และ Dynamic Library | ✅ |
| 40 | โปรเจกต์: Mini Key-Value Store Engine | ✅ |

### Module D — C++ และ OOP (Part 41–55)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 41 | จาก C สู่ C++ | ✅ |
| 42 | C++ I/O Streams และ Namespace | ✅ |
| 43 | Reference และ const | ✅ |
| 44 | ฟังก์ชันใน C++ (Overload/Default/inline) | ✅ |
| 45 | เริ่มต้น Class และ Object | ✅ |
| 46 | Constructor และ Destructor | ✅ |
| 47 | Encapsulation และ Access Specifier | ✅ |
| 48 | Inheritance | ✅ |
| 49 | Polymorphism และ Virtual Function | ✅ |
| 50 | Abstract Class และ Interface | ✅ |
| 51 | Operator Overloading | ✅ |
| 52 | Friend Function และ Friend Class | ✅ |
| 53 | Static Member และ Class Design | ✅ |
| 54 | Exception Handling ใน C++ | ✅ |
| 55 | โปรเจกต์: Library Management System | ✅ |

### Module E — Templates, STL, และ Smart Pointers (Part 56–68)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 56 | Function Template และ Class Template | ✅ |
| 57 | Template Specialization และ Variadic Template | ✅ |
| 58 | แนะนำ STL | ✅ |
| 59 | std::vector และ std::array | ✅ |
| 60 | std::list, std::deque, std::forward_list | ✅ |
| 61 | std::map, std::set, unordered_map/set | ✅ |
| 62 | STL Iterator แบบเจาะลึก | ✅ |
| 63 | STL Algorithm แบบเจาะลึก | ✅ |
| 64 | Function Object, Lambda, std::function | ✅ |
| 65 | std::string และ string_view/Regex | ✅ |
| 66 | Custom Allocator ใน STL | ✅ |
| 67 | Smart Pointer (unique/shared/weak_ptr) | ✅ |
| 68 | RAII และ Resource Management | ✅ |

### Module F — Modern C++11/14/17/20/23 (Part 69–80)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 69 | ภาพรวม C++11 | ✅ |
| 70 | Move Semantics และ Rvalue Reference | ✅ |
| 71 | constexpr และ Compile-Time Programming | ✅ |
| 72 | ฟีเจอร์ C++14/17 | ✅ |
| 73 | C++20 Concepts | ✅ |
| 74 | C++20 Ranges | ✅ |
| 75 | C++20 Coroutines | ✅ |
| 76 | C++20 Modules | ✅ |
| 77 | ภาพรวม C++23 | ✅ |
| 78 | Metaprogramming (SFINAE/type_traits) | ✅ |
| 79 | CRTP และ Template Pattern ขั้นสูง | ✅ |
| 80 | Modern C++ Best Practice & Core Guidelines | ✅ |

### Module G — Concurrency และ Performance (Part 81–90)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 81 | std::thread และ Concurrency | ✅ |
| 82 | std::mutex, std::atomic, Memory Model | ✅ |
| 83 | std::future/async/promise | ✅ |
| 84 | Lock-Free Programming เบื้องต้น | ✅ |
| 85 | Parallel Algorithm (Execution Policy) | ✅ |
| 86 | Performance Profiling (perf/gprof) | ✅ |
| 87 | Cache-Friendly Code & Data-Oriented Design | ✅ |
| 88 | SIMD และ Vectorization | ✅ |
| 89 | Compiler Optimization และ Assembly | ✅ |
| 90 | Benchmarking ด้วย Google Benchmark | ✅ |

### Module H — Build, Test, และ Tooling (Part 91–98)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 91 | CMake ตั้งแต่พื้นฐานถึงขั้นสูง | ✅ |
| 92 | Package Management (Conan/vcpkg) | ✅ |
| 93 | Unit Testing (Google Test/Catch2) | ✅ |
| 94 | CI สำหรับ C++ ด้วย GitHub Actions | ✅ |
| 95 | Static Analysis (clang-tidy/cppcheck) | ✅ |
| 96 | Sanitizer และ Code Coverage | ✅ |
| 97 | Design Pattern ใน C++ (Creational) | ✅ |
| 98 | Design Pattern ใน C++ (Structural/Behavioral) | ✅ |

### Module I — Web Development with C/C++ (Part 99–112)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 99  | ภาพรวม Web Development ด้วย C/C++ | ✅ |
| 100 | สร้าง Raw HTTP Server จาก Socket | ✅ |
| 101 | Multi-threaded HTTP Server + Thread Pool | ✅ |
| 102 | Crow Framework: พื้นฐานและ REST API แรก | ✅ |
| 103 | Crow ขั้นสูง: Middleware/Routing/JSON | ✅ |
| 104 | Pistache Framework | ✅ |
| 105 | เชื่อมต่อ SQLite ด้วย C++ | ✅ |
| 106 | เชื่อมต่อ PostgreSQL/MySQL ด้วย C++ | ✅ |
| 107 | การจัดการ JSON ด้วย nlohmann/json | ✅ |
| 108 | โปรเจกต์: REST API CRUD ครบวงจร | ✅ |
| 109 | WebSocket Programming ด้วย C++ | ✅ |
| 110 | Authentication และ Security (JWT/Hashing) | ✅ |
| 111 | Server-Side Rendering ด้วย C++ | ✅ |
| 112 | Deploy: Docker, Nginx, Production Server | ✅ |

### Module J — Professional Practice (Part 113–120)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 113 | Clean Code และ Code Review Practice | ✅ |
| 114 | Software Architecture สำหรับระบบใหญ่ | ✅ |
| 115 | Security ใน C/C++ และ Secure Coding | ✅ |
| 116 | Cross-Platform Development | ✅ |
| 117 | ภาพรวม Embedded Systems Programming | ✅ |
| 118 | พื้นฐาน Game Development ด้วย C++ | ✅ |
| 119 | เตรียมสัมภาษณ์งาน: DS&A สไตล์ LeetCode | ✅ |
| 120 | เส้นทางอาชีพและ Open Source | ✅ |

### Module K — Capstone Projects และ Final Summary (Part 121–125)

| Part | หัวข้อ | สถานะ |
|---|---|---|
| 121 | Capstone 1: E-Commerce Backend API | ✅ |
| 122 | Capstone 2: Real-time Chat Server | ✅ |
| 123 | Capstone 3: Distributed Key-Value Store | ✅ |
| 124 | Capstone 4: Web Framework ของตัวเอง | ✅ |
| 125 | บทสรุปหลักสูตรและ Roadmap ต่อยอด | ✅ |

## โครงสร้างโฟลเดอร์

```
c_plus_course/
├── README.md              # ไฟล์นี้ - จุดเริ่มต้นและสถานะความคืบหน้า
├── CURRICULUM.md           # โครงสร้างหลักสูตรทั้งหมด 125 Part
├── parts/                  # เนื้อหาแต่ละ Part (part-001-*.md ... part-125-*.md)
└── code/                   # โค้ดตัวอย่าง/โปรเจกต์เสริมที่ยาวเกินจะฝังใน Markdown
```

## License

เนื้อหานี้จัดทำขึ้นเพื่อการศึกษา ใช้เรียนรู้และแจกจ่ายต่อได้อย่างอิสระ
