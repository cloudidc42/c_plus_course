# หลักสูตร C / C++ ฉบับสมบูรณ์ — จากศูนย์สู่ระดับโลก (Step 1–1000)

หลักสูตรนี้ออกแบบให้เดินตาม **Step 1 ถึง Step 1000** โดยแบ่งเป็น **125 Part**
(แต่ละ Part ครอบคลุมประมาณ 8 Step) จัดกลุ่มเป็น 11 Module ใหญ่ ไล่ระดับตั้งแต่
เขียนโปรแกรมไม่เป็นเลย ไปจนถึงระดับ Senior/Staff Engineer ที่ทำงานสาย C/C++
และพัฒนาเว็บแอปพลิเคชันด้วย C/C++ ได้จริงในระดับ Production

> สถานะการเขียน: ดูสถานะล่าสุดของแต่ละ Part ได้ที่ [`README.md`](./README.md)
> ไฟล์เนื้อหาจริงอยู่ในโฟลเดอร์ [`parts/`](./parts)

---

## โครงสร้างภาพรวม

| Module | ชื่อ | Part | Step |
|---|---|---|---|
| A | รากฐานภาษา C (Foundations) | 1–10 | 1–80 |
| B | C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม | 11–25 | 81–200 |
| C | Systems Programming ด้วย C บน Linux | 26–40 | 201–320 |
| D | เริ่มต้น C++ & OOP | 41–55 | 321–440 |
| E | Templates, Generic Programming & STL | 56–68 | 441–544 |
| F | Modern C++ (C++11 → C++23) | 69–80 | 545–640 |
| G | Concurrency & Performance Engineering | 81–90 | 641–720 |
| H | Build Systems, Testing & Tooling | 91–98 | 721–784 |
| I | Web Development ด้วย C/C++ | 99–112 | 785–896 |
| J | Professional & World-Class Practices | 113–120 | 897–960 |
| K | Capstone Projects & บทสรุป | 121–125 | 961–1000 |

---

## Module A — รากฐานภาษา C (Foundations of C)

1. เริ่มต้นกับภาษา C: ประวัติศาสตร์, ทำไมต้อง C, ติดตั้ง Toolchain (GCC/Clang, VS Code), คอมไพล์โปรแกรมแรก, กระบวนการ Compile-Link-Run
2. ตัวแปร ชนิดข้อมูล และตัวดำเนินการ: int/float/double/char, การประกาศตัวแปร, Type Conversion, Operator ทุกประเภท
3. การรับส่งข้อมูลเชิงลึก: printf/scanf, Format Specifier ครบทุกแบบ, Buffer และ stdin/stdout/stderr
4. โครงสร้างควบคุมเงื่อนไข: if/else if/else, switch-case, Ternary Operator, การออกแบบเงื่อนไขที่อ่านง่าย
5. การวนซ้ำ: for, while, do-while, break/continue, Loop Pattern ที่ใช้บ่อยในงานจริง
6. ฟังก์ชันในภาษา C: การประกาศ/นิยาม, Pass by Value, Recursion, Scope และ Storage Class
7. Array และ String ใน C: Array 1 มิติ, การจัดการ String ด้วยมือ, ฟังก์ชันใน string.h
8. Pointer พื้นฐาน (ตอนที่ 1): แนวคิด Address/Value, Pointer Declaration, Pointer กับ Array
9. Pointer ขั้นสูง (ตอนที่ 2): Pointer Arithmetic, Pointer to Pointer, Function Pointer, Const Pointer
10. Struct, Union, Enum และ typedef: การออกแบบ Data Type ของตัวเอง

## Module B — C ระดับกลาง & โครงสร้างข้อมูล/อัลกอริทึม

11. Dynamic Memory Allocation: malloc/calloc/realloc/free, Memory Leak, Dangling Pointer
12. Array หลายมิติ และ Pointer-to-Pointer: Matrix, Dynamic 2D Array
13. File I/O ใน C: fopen/fread/fwrite/fprintf, Text vs Binary File, การอ่านไฟล์ขนาดใหญ่
14. Preprocessor และ Macro: #define, Macro Function, Conditional Compilation, Header Guard
15. Bit Manipulation: Bitwise Operator, Bit Flag, Bit Field ใน Struct, การประยุกต์ใช้จริง
16. Error Handling ใน C: errno, การออกแบบ Return Code, assert, Defensive Programming
17. Modular Programming: Header File, Compilation Unit, Linkage (extern/static), การออกแบบ Module
18. Makefile และ Build Automation เบื้องต้น: เขียน Makefile ตั้งแต่ง่ายถึงมี Pattern Rule
19. Linked List: Singly, Doubly, Circular Linked List พร้อม Implementation เต็มรูปแบบ
20. Stack และ Queue ด้วย C: Array-based และ Linked List-based, Application จริง (Undo, BFS)
21. Tree เบื้องต้น: Binary Tree, Binary Search Tree, Traversal (Inorder/Preorder/Postorder)
22. Hash Table ด้วย C: Hash Function, Collision Resolution (Chaining, Open Addressing)
23. Sorting Algorithm: Bubble, Selection, Insertion, Merge, Quick Sort พร้อมวิเคราะห์ความซับซ้อน
24. Searching Algorithm และ Big-O: Linear/Binary Search, การวิเคราะห์ Time/Space Complexity
25. Recursion ขั้นสูงและ Dynamic Programming เบื้องต้น: Memoization, Tabulation, ตัวอย่างโจทย์คลาสสิก

## Module C — Systems Programming ด้วย C บน Linux

26. พื้นฐานระบบปฏิบัติการสำหรับโปรแกรมเมอร์: Process, Memory Layout, Kernel vs User Space
27. Process Management: fork(), exec(), wait(), Zombie/Orphan Process
28. Signal และ Signal Handling: signal(), sigaction(), การจัดการ Ctrl+C, SIGSEGV
29. Inter-Process Communication: Pipe, FIFO (Named Pipe)
30. Shared Memory และ Message Queue (System V / POSIX IPC)
31. POSIX Threads (pthreads) เบื้องต้น: การสร้าง/รอ Thread, Thread Argument
32. Thread Synchronization: Mutex, Semaphore, Condition Variable, Deadlock/Race Condition
33. Socket Programming เบื้องต้น: TCP Socket, Client-Server Model แรก
34. Socket Programming ขั้นสูง: UDP, select(), poll(), epoll() สำหรับ I/O Multiplexing
35. โปรเจกต์: สร้าง TCP Chat Server/Client แบบ Multi-client ด้วย C
36. Memory-Mapped File ด้วย mmap: การใช้งานจริงและประโยชน์
37. การ Debug ด้วย GDB: Breakpoint, Watch, Backtrace, Core Dump Analysis
38. Valgrind และการตรวจจับ Memory Leak/Undefined Behavior
39. Static Library (.a) และ Dynamic/Shared Library (.so): การสร้างและใช้งาน
40. โปรเจกต์ Module C: สร้าง Mini Key-Value Store Database Engine ด้วย C ล้วน

## Module D — เริ่มต้น C++ และ OOP

41. จาก C สู่ C++: ความแตกต่างสำคัญ, ทำไม C++ ถึงทรงพลัง, คอมไพล์โปรแกรม C++ แรก
42. C++ I/O Streams: cin/cout/cerr, Namespace, std::string เบื้องต้น
43. Reference และ const: ความแตกต่างจาก Pointer, Reference Parameter, const Correctness
44. ฟังก์ชันใน C++: Function Overloading, Default Argument, inline Function
45. เริ่มต้น Class และ Object: การออกแบบ Class แรก, Member Variable/Function
46. Constructor และ Destructor: ทุกประเภทของ Constructor, RAII เบื้องต้น
47. Encapsulation และ Access Specifier: public/private/protected, Getter/Setter ที่ดี
48. Inheritance: Single, Multilevel, Multiple Inheritance, Diamond Problem
49. Polymorphism และ Virtual Function: Virtual Table, Override, Dynamic Dispatch
50. Abstract Class และ Interface: Pure Virtual Function, การออกแบบ Interface ที่ดี
51. Operator Overloading: overload operator ทุกประเภทอย่างถูกวิธี
52. Friend Function และ Friend Class: การใช้งานอย่างเหมาะสม
53. Static Member และการออกแบบระดับ Class: Static Data/Function, Singleton เบื้องต้น
54. Exception Handling ใน C++: try/catch/throw, Exception Hierarchy, Custom Exception
55. โปรเจกต์ Module D: ระบบจัดการห้องสมุด (Library Management System) แบบ OOP เต็มรูปแบบ

## Module E — Templates, Generic Programming และ STL

56. Function Template และ Class Template: การเขียนโค้ด Generic
57. Template Specialization และ Variadic Template
58. แนะนำ STL: ภาพรวม Container/Iterator/Algorithm/Functor
59. std::vector และ std::array แบบเจาะลึก
60. std::list, std::deque, std::forward_list: เมื่อไหร่ควรใช้อะไร
61. std::map, std::set, std::unordered_map/unordered_set
62. STL Iterator แบบเจาะลึก: ประเภทของ Iterator, Custom Iterator
63. STL Algorithm (<algorithm>) แบบเจาะลึก: sort, find, transform, accumulate ฯลฯ
64. Function Object, Lambda Expression และ std::function
65. std::string และการจัดการ String ขั้นสูง: string_view, Regex เบื้องต้น
66. Custom Allocator และการจัดการหน่วยความจำใน STL
67. Smart Pointer: unique_ptr, shared_ptr, weak_ptr แบบเจาะลึก
68. RAII และ Resource Management Pattern ระดับ Production

## Module F — Modern C++ (C++11 ถึง C++23)

69. ภาพรวม C++11: auto, decltype, range-based for, nullptr, enum class
70. Move Semantics และ Rvalue Reference: std::move, Move Constructor/Assignment
71. constexpr และ Compile-Time Programming
72. ฟีเจอร์ C++14/17: Structured Bindings, if constexpr, std::optional/variant/any
73. C++20 Concepts: การจำกัด Template ด้วย Concept
74. C++20 Ranges: การเขียน Pipeline แบบ Functional
75. C++20 Coroutines: co_await, co_yield, co_return
76. C++20 Modules: อนาคตของการจัดการ Header
77. ภาพรวมฟีเจอร์ใหม่ C++23
78. Metaprogramming ด้วย Template: SFINAE, type_traits, Concept-based Dispatch
79. CRTP และ Pattern การใช้ Template ขั้นสูง
80. Modern C++ Best Practice และ C++ Core Guidelines

## Module G — Concurrency และ Performance Engineering

81. std::thread และ Concurrency ใน Modern C++
82. std::mutex, std::atomic และ C++ Memory Model
83. Async Programming: std::future, std::async, std::promise, std::packaged_task
84. Lock-Free Programming เบื้องต้น
85. Parallel Algorithm (C++17 Execution Policy)
86. Performance Optimization: การ Profile ด้วย perf/gprof
87. Cache-Friendly Code และ Data-Oriented Design
88. SIMD และ Vectorization เบื้องต้น
89. ทำความเข้าใจ Compiler Optimization และ Assembly Output
90. Benchmarking ด้วย Google Benchmark

## Module H — Build Systems, Testing และ Tooling

91. CMake ตั้งแต่พื้นฐานถึงขั้นสูง
92. Package Management: Conan และ vcpkg
93. Unit Testing ด้วย Google Test และ Catch2
94. Continuous Integration สำหรับ C++ ด้วย GitHub Actions
95. Static Analysis: clang-tidy, cppcheck, clang-format
96. Sanitizer และ Code Coverage: ASan, UBSan, TSan, gcov/lcov
97. Design Pattern ใน C++ (ตอนที่ 1): Creational Patterns
98. Design Pattern ใน C++ (ตอนที่ 2): Structural และ Behavioral Patterns

## Module I — Web Development ด้วย C/C++

99. ภาพรวม Web Development ด้วย C/C++: CGI, FastCGI, Embedded HTTP Server, สถาปัตยกรรม
100. สร้าง Raw HTTP Server จาก Socket ด้วยมือ (From Scratch)
101. สร้าง Multi-threaded HTTP Server พร้อม Thread Pool
102. Crow Framework: พื้นฐานและ REST API แรก
103. Crow Framework ขั้นสูง: Middleware, Routing, JSON Response
104. Pistache Framework: ทางเลือกสำหรับ REST API ระดับ Production
105. เชื่อมต่อฐานข้อมูล SQLite ด้วย C++
106. เชื่อมต่อฐานข้อมูล PostgreSQL/MySQL ด้วย libpqxx/MySQL Connector
107. การจัดการ JSON ด้วย nlohmann/json
108. โปรเจกต์: สร้าง REST API Backend แบบ CRUD ครบวงจร
109. WebSocket Programming ด้วย C++
110. Authentication และ Security: JWT, Password Hashing (bcrypt), HTTPS
111. Server-Side Rendering: การ Serve HTML/Template จาก C++
112. Deploy C++ Web Application: Docker, Nginx Reverse Proxy, Linux Production Server

## Module J — Professional และ World-Class Practices

113. Clean Code และ Code Review Practice สำหรับ C/C++
114. Software Architecture สำหรับระบบ C++ ขนาดใหญ่
115. Security ใน C/C++: ช่องโหว่ที่พบบ่อยและ Secure Coding Practice
116. Cross-Platform Development: Windows/Linux/macOS
117. ภาพรวม Embedded Systems Programming ด้วย C/C++
118. พื้นฐาน Game Development ด้วย C++ (SDL2/OpenGL)
119. เตรียมสัมภาษณ์งาน: Data Structures & Algorithms สไตล์ LeetCode ด้วย C++
120. เส้นทางอาชีพ: จาก Junior สู่ Senior/Staff Engineer และการมีส่วนร่วมกับ Open Source

## Module K — Capstone Projects และบทสรุป

121. Capstone 1: E-Commerce Backend API เต็มรูปแบบ (C++ + Crow + PostgreSQL + Docker)
122. Capstone 2: Real-time Multiplayer Chat Server (Socket + WebSocket + Multi-threading)
123. Capstone 3: Distributed Key-Value Store (Networking + Concurrency + Persistence)
124. Capstone 4: สร้าง Web Framework ของตัวเองด้วย C++ ตั้งแต่ศูนย์
125. บทสรุปหลักสูตร: Roadmap สู่ระดับ World-Class Engineer และแหล่งเรียนรู้ต่อยอด

---

## แนวทางการเขียนเนื้อหาแต่ละ Part

ทุก Part ต้องมีองค์ประกอบดังนี้เป็นอย่างน้อย:

1. **หัวข้อและเป้าหมายการเรียนรู้ (Learning Objectives)**
2. **คำอธิบายเชิงทฤษฎี** ที่เข้าใจง่าย พร้อมเปรียบเทียบ/ยกตัวอย่างในโลกจริง
3. **โค้ดตัวอย่างที่คอมไพล์และรันได้จริง 100%** พร้อมคำอธิบายทีละบรรทัด/ทีละบล็อกสำคัญ
4. **กรณีศึกษา/ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)**
5. **แบบฝึกหัดท้ายบท** พร้อมแนวทางเฉลย
6. **สรุปท้ายบท** และลิงก์ไปยัง Part ถัดไป/ก่อนหน้า

ความยาวเป้าหมายต่อ Part: **500–3000+ บรรทัด Markdown**
