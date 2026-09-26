# Part 115: Security ใน C/C++ และ Secure Coding Practice (Step 913–920)

> Module J — Professional Practice | Part 115 จาก 125
> Step ที่ครอบคลุมใน Part นี้: Step 913–920
> Part ก่อนหน้า: [Part 114 — Software Architecture สำหรับระบบใหญ่](./part-114-software-architecture.md) | Part ถัดไป: [Part 116 — Cross-Platform Development](./part-116-cross-platform.md)

---

## บทนำ: ทำไม Security ใน C/C++ ถึงสำคัญมาก

ภาษา C และ C++ ถูกใช้งานในระบบที่สำคัญที่สุดของโลก ไม่ว่าจะเป็น operating system, embedded firmware, browser engine, database, network stack หรือแม้แต่ระบบควบคุมในยานพาหนะ เพราะความเร็วและความสามารถในการควบคุม hardware โดยตรง

แต่ด้วยพลังนี้มาพร้อมความรับผิดชอบ — C/C++ ไม่ได้ปกป้องโปรแกรมเมอร์จากความผิดพลาดโดยอัตโนมัติ ความผิดพลาดที่ดูเล็กน้อยในการจัดการ memory หรือ string อาจก่อให้เกิด bug ที่นำไปสู่ช่องโหว่ด้านความปลอดภัยได้

Part นี้จะสอนวิธี **เขียนโค้ดที่ปลอดภัย** ตั้งแต่การเข้าใจว่าอะไรทำให้โค้ดเสี่ยง ไปจนถึงเครื่องมือที่ช่วยตรวจจับปัญหาก่อนที่จะ ship code

---

## Step 913: ทำไม C/C++ ถึงเสี่ยงต่อ Vulnerability

### ลักษณะของ C/C++ ที่แตกต่างจากภาษาสมัยใหม่

ภาษาอย่าง Java, Python หรือ Rust มีการป้องกัน memory errors หลายอย่างโดยอัตโนมัติ แต่ C/C++ ไม่มี เพราะถูกออกแบบมาเพื่อ performance และ control สูงสุด

```c
// ตัวอย่างง่ายๆ: C ไม่มี bounds checking
#include <stdio.h>

int main(void) {
    int array[5] = {1, 2, 3, 4, 5};
    
    // ใน Java/Python: จะเกิด IndexOutOfBoundsException ทันที
    // ใน C: compiler ไม่บังคับ — พฤติกรรม undefined
    // อย่าเขียนแบบนี้ในโค้ดจริง:
    // int x = array[10];  // อ่านข้อมูลที่ไม่ใช่ของ array
    
    // วิธีที่ถูกต้อง: ตรวจก่อนเสมอ
    int index = 10;
    if (index >= 0 && index < 5) {
        printf("array[%d] = %d\n", index, array[index]);
    } else {
        printf("Error: index %d is out of bounds (valid: 0-4)\n", index);
    }
    
    return 0;
}
```

### ปัญหาหลักของ C/C++

| ปัญหา | คำอธิบาย | ผลที่ตามมา |
|-------|----------|-----------|
| ไม่มี bounds checking | array access ไม่ถูกตรวจ | Buffer overflow |
| Manual memory management | programmer ต้อง free เอง | Use-after-free, memory leak |
| Pointer arithmetic | คำนวณ address ได้โดยตรง | Out-of-bounds read/write |
| ไม่มี type safety เต็มรูปแบบ | cast แบบไหนก็ได้ | Type confusion |
| Integer ไม่ safe | overflow/underflow ไม่ถูกตรวจ | Integer overflow bugs |

### ทำไมต้องเรียนรู้เรื่องนี้

ตามสถิติจาก National Vulnerability Database (NVD) หลายปีที่ผ่านมา ช่องโหว่ที่พบบ่อยที่สุดใน software ที่เขียนด้วย C/C++ ได้แก่:

- **Buffer Overflow** — อ่านหรือเขียนข้อมูลเกิน boundary ของ buffer
- **Use-After-Free** — ใช้งาน pointer หลังจาก free memory แล้ว
- **Integer Overflow** — ค่า integer ล้นออกนอก range ที่รองรับ
- **Format String Bug** — ส่ง format string ที่ไม่ปลอดภัยให้ printf/scanf
- **NULL Pointer Dereference** — dereference pointer ที่เป็น NULL

เป้าหมายของ Part นี้คือสอนให้เขียนโค้ดที่ **หลีกเลี่ยงปัญหาเหล่านี้ตั้งแต่ต้น**

---

## Step 914: Buffer Overflow — เข้าใจและป้องกัน

### Buffer Overflow คืออะไร

Buffer Overflow เกิดขึ้นเมื่อโปรแกรมเขียนข้อมูลเกินพื้นที่ที่จัดสรรไว้ให้ buffer ทำให้ข้อมูลทับ memory ส่วนอื่น

### โค้ดที่เสี่ยง (อย่าเขียนแบบนี้)

```c
#include <stdio.h>
#include <string.h>

// ฟังก์ชันนี้มีปัญหา — ห้ามใช้ในโค้ดจริง
void unsafe_copy(const char *input) {
    char buffer[64];
    // strcpy ไม่ตรวจความยาว — ถ้า input ยาวกว่า 63 ตัวอักษร จะเขียนทับ memory ข้างๆ
    // strcpy(buffer, input);  // อันตราย! อย่าใช้
    (void)input;  // suppress warning ในตัวอย่างนี้
    (void)buffer;
    printf("ห้ามใช้ strcpy โดยไม่ตรวจ bounds\n");
}

int main(void) {
    unsafe_copy("example");
    return 0;
}
```

### โค้ดที่ปลอดภัย

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

// วิธีที่ 1: ใช้ strncpy และตรวจสอบ
void safe_copy_v1(const char *input, char *dest, size_t dest_size) {
    if (input == NULL || dest == NULL || dest_size == 0) {
        return;
    }
    // strncpy copy ได้สูงสุด dest_size-1 ตัวอักษร
    strncpy(dest, input, dest_size - 1);
    dest[dest_size - 1] = '\0';  // ต้อง null-terminate เอง
}

// วิธีที่ 2: ใช้ snprintf — ดีกว่า strncpy มาก
void safe_copy_v2(const char *input, char *dest, size_t dest_size) {
    if (input == NULL || dest == NULL || dest_size == 0) {
        return;
    }
    // snprintf จัดการ null-termination ให้อัตโนมัติ
    int written = snprintf(dest, dest_size, "%s", input);
    if (written < 0 || (size_t)written >= dest_size) {
        // ข้อมูลถูก truncate — ตัดสินใจว่าจะ handle อย่างไร
        fprintf(stderr, "Warning: input was truncated\n");
    }
}

// วิธีที่ 3: ตรวจความยาวก่อน
int safe_copy_v3(const char *input, char *dest, size_t dest_size) {
    if (input == NULL || dest == NULL || dest_size == 0) {
        return -1;
    }
    size_t input_len = strlen(input);
    if (input_len >= dest_size) {
        fprintf(stderr, "Error: input too long (%zu chars, max %zu)\n",
                input_len, dest_size - 1);
        return -1;
    }
    memcpy(dest, input, input_len + 1);  // +1 สำหรับ null terminator
    return 0;
}

int main(void) {
    char buffer[64];
    const char *long_input = "This is a test string that fits in 64 chars buffer.";
    const char *very_long = "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
                            "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA";
    
    // ทดสอบ v2 กับ input ที่พอดี
    safe_copy_v2(long_input, buffer, sizeof(buffer));
    printf("v2 result: %s\n", buffer);
    
    // ทดสอบ v2 กับ input ที่ยาวเกิน — จะ truncate และแจ้งเตือน
    safe_copy_v2(very_long, buffer, sizeof(buffer));
    printf("v2 truncated: %s\n", buffer);
    
    // ทดสอบ v3
    if (safe_copy_v3(long_input, buffer, sizeof(buffer)) == 0) {
        printf("v3 result: %s\n", buffer);
    }
    if (safe_copy_v3(very_long, buffer, sizeof(buffer)) != 0) {
        printf("v3: rejected input that was too long\n");
    }
    
    return 0;
}
```

### Buffer Overflow ใน String Formatting

```c
#include <stdio.h>
#include <string.h>

// ปัญหา: sprintf ไม่ตรวจ bounds
void format_message_unsafe(const char *name, int score) {
    char message[64];
    // sprintf(message, "Player %s scored %d points!", name, score);
    // ถ้า name ยาว message จะ overflow
    (void)name; (void)score; (void)message;
    printf("ห้ามใช้ sprintf โดยไม่ตรวจ bounds\n");
}

// วิธีที่ปลอดภัย: ใช้ snprintf เสมอ
int format_message_safe(char *buf, size_t buf_size,
                        const char *name, int score) {
    if (buf == NULL || buf_size == 0 || name == NULL) {
        return -1;
    }
    int result = snprintf(buf, buf_size,
                          "Player %s scored %d points!", name, score);
    if (result < 0) {
        return -1;  // encoding error
    }
    if ((size_t)result >= buf_size) {
        return -2;  // output was truncated
    }
    return result;  // จำนวน character ที่เขียน
}

int main(void) {
    char message[64];
    
    int ret = format_message_safe(message, sizeof(message), "Alice", 9999);
    if (ret > 0) {
        printf("%s\n", message);
    }
    
    // ทดสอบกับชื่อยาวมาก
    const char *long_name = "VeryLongPlayerNameThatExceedsBufferLimit";
    ret = format_message_safe(message, sizeof(message), long_name, 100);
    if (ret == -2) {
        printf("Message was truncated — handle this appropriately\n");
    }
    
    return 0;
}
```

### หลักการป้องกัน Buffer Overflow

1. **ใช้ `snprintf` แทน `sprintf` เสมอ**
2. **ใช้ `strncpy` หรือ `strlcpy` แทน `strcpy`** (และต้อง null-terminate เอง)
3. **ตรวจสอบ return value** ของ string functions
4. **กำหนดขนาด buffer ด้วย `sizeof`** ไม่ใช่ magic number
5. **ใน C++: ใช้ `std::string`** แทน character arrays เมื่อเป็นไปได้

```cpp
// C++ style — ปลอดภัยกว่ามาก
#include <iostream>
#include <string>
#include <sstream>

std::string format_message_cpp(const std::string& name, int score) {
    // std::string จัดการ memory อัตโนมัติ ไม่มี overflow
    std::ostringstream oss;
    oss << "Player " << name << " scored " << score << " points!";
    return oss.str();
}

int main() {
    std::string msg = format_message_cpp("Alice", 9999);
    std::cout << msg << "\n";
    
    // แม้ชื่อยาวมาก ก็ไม่ overflow
    std::string long_name(1000, 'A');
    msg = format_message_cpp(long_name, 100);
    std::cout << "Long name handled safely, length: " << msg.size() << "\n";
    
    return 0;
}
```

---

## Step 915: Use-After-Free และ Double-Free

### Use-After-Free คืออะไร

Use-After-Free (UAF) เกิดขึ้นเมื่อโปรแกรม `free()` memory แล้วยังคงใช้งาน pointer นั้นต่อ Memory ที่ free แล้วอาจถูกนำไปใช้ที่อื่น ทำให้อ่านข้อมูลผิด หรือเขียนทับข้อมูลของส่วนอื่น

### Double-Free คืออะไร

Double-Free เกิดขึ้นเมื่อ `free()` memory ชิ้นเดียวกันสองครั้ง ทำให้ memory allocator เสียหาย

### โค้ดที่เสี่ยง (อย่าทำแบบนี้)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

struct Node {
    int value;
    struct Node *next;
};

// ตัวอย่างปัญหา UAF — อย่าทำแบบนี้
void demonstrate_uaf_problem(void) {
    struct Node *node = (struct Node *)malloc(sizeof(struct Node));
    if (node == NULL) return;
    
    node->value = 42;
    
    free(node);
    // node ยังชี้ไปยัง address เดิม แต่ memory ถูก free แล้ว
    // การอ่านหลังจากนี้เป็น undefined behavior:
    // printf("%d\n", node->value);  // อันตราย! อย่าทำ
    // การเขียนหลังจากนี้ก็อันตราย:
    // node->value = 100;  // อันตราย! อย่าทำ
    
    printf("ตัวอย่างนี้แสดงว่า อย่าใช้ pointer หลัง free\n");
}

int main(void) {
    demonstrate_uaf_problem();
    return 0;
}
```

### โค้ดที่ปลอดภัย: ตั้ง NULL หลัง free

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Macro ที่ช่วยป้องกัน — free และ set NULL ในครั้งเดียว
#define SAFE_FREE(ptr) do { free(ptr); (ptr) = NULL; } while(0)

struct Node {
    int value;
    char *data;
    struct Node *next;
};

struct Node *create_node(int value, const char *data) {
    struct Node *node = (struct Node *)malloc(sizeof(struct Node));
    if (node == NULL) {
        return NULL;
    }
    
    node->value = value;
    node->next = NULL;
    
    if (data != NULL) {
        size_t data_len = strlen(data);
        node->data = (char *)malloc(data_len + 1);
        if (node->data == NULL) {
            free(node);
            return NULL;
        }
        memcpy(node->data, data, data_len + 1);
    } else {
        node->data = NULL;
    }
    
    return node;
}

void destroy_node(struct Node **node_ptr) {
    if (node_ptr == NULL || *node_ptr == NULL) {
        return;  // ป้องกัน double-free และ NULL dereference
    }
    
    struct Node *node = *node_ptr;
    
    // free ข้อมูลภายในก่อน
    SAFE_FREE(node->data);
    
    // free node และ set pointer เป็น NULL
    SAFE_FREE(*node_ptr);
    // *node_ptr ตอนนี้เป็น NULL แล้ว
}

int main(void) {
    struct Node *node = create_node(42, "Hello");
    if (node == NULL) {
        fprintf(stderr, "Failed to create node\n");
        return 1;
    }
    
    printf("Node value: %d, data: %s\n", node->value, node->data);
    
    // ลบ node — ส่ง address ของ pointer เพื่อให้ set NULL ได้
    destroy_node(&node);
    
    // ตอนนี้ node เป็น NULL แล้ว — ปลอดภัยที่จะตรวจ
    if (node == NULL) {
        printf("Node was safely freed and pointer is NULL\n");
    }
    
    // การเรียก destroy_node อีกครั้งจะปลอดภัย เพราะตรวจ NULL
    destroy_node(&node);  // ไม่เกิด double-free
    printf("Double-free was safely prevented\n");
    
    return 0;
}
```

### ใช้ Smart Pointer ใน C++ เพื่อป้องกัน UAF และ Double-Free

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <vector>

struct Node {
    int value;
    std::string data;
    
    Node(int v, std::string d) : value(v), data(std::move(d)) {
        std::cout << "Node created: " << value << "\n";
    }
    ~Node() {
        std::cout << "Node destroyed: " << value << "\n";
    }
};

// ใช้ unique_ptr — ownership ชัดเจน, auto-free เมื่อออกจาก scope
void demonstrate_unique_ptr() {
    std::cout << "--- unique_ptr demo ---\n";
    
    auto node = std::make_unique<Node>(42, "Hello");
    std::cout << "Node value: " << node->value << "\n";
    
    // ไม่ต้อง delete เอง — จะ free อัตโนมัติเมื่อ node ออกจาก scope
    // ไม่มีโอกาส UAF หรือ double-free
}

// ใช้ shared_ptr — หลาย owner ได้
void demonstrate_shared_ptr() {
    std::cout << "--- shared_ptr demo ---\n";
    
    auto node1 = std::make_shared<Node>(100, "Shared");
    {
        auto node2 = node1;  // ทั้ง node1 และ node2 ชี้ไปที่เดียวกัน
        std::cout << "ref count: " << node1.use_count() << "\n";  // 2
        std::cout << "node2 value: " << node2->value << "\n";
    }
    // node2 ออกจาก scope แล้ว แต่ node ยังอยู่เพราะ node1 ยังอ้างถึง
    std::cout << "ref count after block: " << node1.use_count() << "\n";  // 1
}

// ใช้ vector ของ unique_ptr
void demonstrate_container_of_pointers() {
    std::cout << "--- container of smart pointers ---\n";
    
    std::vector<std::unique_ptr<Node>> nodes;
    nodes.push_back(std::make_unique<Node>(1, "First"));
    nodes.push_back(std::make_unique<Node>(2, "Second"));
    nodes.push_back(std::make_unique<Node>(3, "Third"));
    
    for (const auto& n : nodes) {
        std::cout << "  value=" << n->value << " data=" << n->data << "\n";
    }
    // nodes ถูก destroy เมื่อออกจาก scope — ทุก Node ถูก free อัตโนมัติ
}

int main() {
    demonstrate_unique_ptr();
    demonstrate_shared_ptr();
    demonstrate_container_of_pointers();
    return 0;
}
```

### กฎการป้องกัน UAF และ Double-Free

1. **ใน C: ตั้ง pointer เป็น `NULL` ทันทีหลัง `free()`**
2. **ตรวจ `NULL` ก่อน dereference pointer เสมอ**
3. **ใน C++: ใช้ smart pointer (`unique_ptr`, `shared_ptr`) แทน raw pointer**
4. **กำหนด ownership ให้ชัดเจน** — ใครเป็นเจ้าของ memory และใครเป็นคน free
5. **ใช้ RAII pattern** — ผูก resource lifecycle เข้ากับ object lifetime

---

## Step 916: Integer Overflow และ Underflow

### Integer Overflow คืออะไร

Integer Overflow เกิดขึ้นเมื่อการคำนวณให้ผลลัพธ์ที่เกิน range ของ type นั้น เช่น `INT_MAX + 1` บนระบบส่วนใหญ่จะ wrap around กลับมาเป็น `INT_MIN`

ปัญหานี้อันตรายเป็นพิเศษเมื่อใช้ผลลัพธ์เพื่อจัดสรร memory หรือตรวจสอบ bounds

### ตัวอย่างและวิธีป้องกัน

```c
#include <stdio.h>
#include <stdlib.h>
#include <limits.h>
#include <stdint.h>

// ตัวอย่าง: การจัดสรร memory ที่ไม่ปลอดภัย
// ถ้า count * element_size overflow จะจัดสรร memory น้อยกว่าที่ต้องการ
void *unsafe_alloc(size_t count, size_t element_size) {
    // อันตราย: count * element_size อาจ overflow
    // void *p = malloc(count * element_size);
    // return p;
    (void)count; (void)element_size;
    return NULL;
}

// วิธีที่ 1: ตรวจ overflow ก่อนคำนวณ
void *safe_alloc_v1(size_t count, size_t element_size) {
    if (element_size != 0 && count > SIZE_MAX / element_size) {
        fprintf(stderr, "Error: allocation size overflow\n");
        return NULL;
    }
    return malloc(count * element_size);
}

// วิธีที่ 2: ใช้ calloc ซึ่งตรวจ overflow ให้อัตโนมัติ
void *safe_alloc_v2(size_t count, size_t element_size) {
    // calloc ตรวจ overflow และ zero-initialize ด้วย
    return calloc(count, element_size);
}

// ตรวจ overflow สำหรับการบวก
int safe_add_int(int a, int b, int *result) {
    // ตรวจก่อนบวก เพื่อหลีกเลี่ยง undefined behavior
    if ((b > 0 && a > INT_MAX - b) ||
        (b < 0 && a < INT_MIN - b)) {
        return -1;  // overflow
    }
    *result = a + b;
    return 0;
}

// ตรวจ overflow สำหรับการคูณ
int safe_multiply_int(int a, int b, int *result) {
    if (a != 0 && b != 0) {
        if ((a > 0 && b > 0 && a > INT_MAX / b) ||
            (a < 0 && b < 0 && a < INT_MAX / b) ||
            (a > 0 && b < 0 && b < INT_MIN / a) ||
            (a < 0 && b > 0 && a < INT_MIN / b)) {
            return -1;  // overflow
        }
    }
    *result = a * b;
    return 0;
}

// ใช้ unsigned arithmetic อย่างระมัดระวัง
// Underflow ของ unsigned: 0u - 1u = UINT_MAX (ไม่ใช่ -1)
void demonstrate_unsigned_pitfall(void) {
    unsigned int a = 5;
    unsigned int b = 10;
    
    // อันตราย: a - b เมื่อ a < b จะไม่ได้ -5 แต่ได้ UINT_MAX - 4
    // if (a - b > 0) { ... }  // ตรวจผิด! ผลลัพธ์เป็น unsigned เสมอ
    
    // วิธีที่ถูก: ตรวจก่อน
    if (a >= b) {
        printf("a - b = %u\n", a - b);
    } else {
        printf("b is larger than a, difference would underflow\n");
    }
}

// ใช้ fixed-size types เมื่อต้องการความแน่นอน
void demonstrate_fixed_types(void) {
    // int32_t รับประกันว่าเป็น 32-bit signed บนทุก platform
    int32_t x = INT32_MAX;
    printf("INT32_MAX = %d\n", x);
    
    // uint64_t สำหรับค่าที่ต้องการ range กว้าง
    uint64_t big = UINT64_MAX;
    printf("UINT64_MAX = %llu\n", (unsigned long long)big);
}

int main(void) {
    // ทดสอบ safe_alloc
    int *arr = (int *)safe_alloc_v2(100, sizeof(int));
    if (arr != NULL) {
        arr[0] = 42;
        printf("safe_alloc: arr[0] = %d\n", arr[0]);
        free(arr);
    }
    
    // ทดสอบ safe arithmetic
    int result = 0;
    if (safe_add_int(INT_MAX, 1, &result) != 0) {
        printf("Addition would overflow — rejected\n");
    }
    if (safe_add_int(100, 200, &result) == 0) {
        printf("100 + 200 = %d\n", result);
    }
    if (safe_multiply_int(1000000, 1000000, &result) != 0) {
        printf("Multiplication would overflow — rejected\n");
    }
    
    demonstrate_unsigned_pitfall();
    demonstrate_fixed_types();
    
    return 0;
}
```

### ใช้ Checked Arithmetic ใน C++

```cpp
#include <iostream>
#include <limits>
#include <stdexcept>
#include <type_traits>

// Template function สำหรับ safe addition
template<typename T>
T safe_add(T a, T b) {
    static_assert(std::is_integral<T>::value, "T must be integral type");
    if constexpr (std::is_signed<T>::value) {
        if ((b > 0 && a > std::numeric_limits<T>::max() - b) ||
            (b < 0 && a < std::numeric_limits<T>::min() - b)) {
            throw std::overflow_error("Integer addition overflow");
        }
    } else {
        // unsigned
        if (a > std::numeric_limits<T>::max() - b) {
            throw std::overflow_error("Integer addition overflow");
        }
    }
    return a + b;
}

// Template function สำหรับ safe multiplication
template<typename T>
T safe_multiply(T a, T b) {
    static_assert(std::is_integral<T>::value, "T must be integral type");
    if (a == 0 || b == 0) return 0;
    if constexpr (std::is_signed<T>::value) {
        if (a > 0 && b > 0 && a > std::numeric_limits<T>::max() / b)
            throw std::overflow_error("Integer multiplication overflow");
        if (a < 0 && b < 0 && a < std::numeric_limits<T>::max() / b)
            throw std::overflow_error("Integer multiplication overflow");
        if (a > 0 && b < 0 && b < std::numeric_limits<T>::min() / a)
            throw std::overflow_error("Integer multiplication overflow");
        if (a < 0 && b > 0 && a < std::numeric_limits<T>::min() / b)
            throw std::overflow_error("Integer multiplication overflow");
    } else {
        if (a > std::numeric_limits<T>::max() / b)
            throw std::overflow_error("Integer multiplication overflow");
    }
    return a * b;
}

int main() {
    try {
        int32_t x = safe_add<int32_t>(100, 200);
        std::cout << "100 + 200 = " << x << "\n";
        
        int32_t y = safe_add<int32_t>(std::numeric_limits<int32_t>::max(), 1);
        std::cout << "This should not print\n";
        (void)y;
    } catch (const std::overflow_error& e) {
        std::cout << "Caught overflow: " << e.what() << "\n";
    }
    
    try {
        int32_t z = safe_multiply<int32_t>(1000, 2000);
        std::cout << "1000 * 2000 = " << z << "\n";
        
        int32_t w = safe_multiply<int32_t>(1000000, 1000000);
        std::cout << "This should not print\n";
        (void)w;
    } catch (const std::overflow_error& e) {
        std::cout << "Caught overflow: " << e.what() << "\n";
    }
    
    return 0;
}
```

---

## Step 917: Format String Vulnerabilities — ป้องกันได้ง่าย

### Format String Bug คืออะไร

Format String Bug เกิดเมื่อผ่าน user input โดยตรงเป็น format string ให้กับ `printf`, `fprintf`, `sprintf` เป็นต้น โดยไม่ได้ระบุ format specifier

### ตัวอย่างที่ปลอดภัย

```c
#include <stdio.h>
#include <string.h>

// กฎง่ายๆ: ใช้ format string literal เสมอ — ไม่ใช่ตัวแปร

// อันตราย: อย่าทำแบบนี้
void print_message_unsafe(const char *message) {
    // printf(message);  // อันตราย! ถ้า message มี %s, %n จะเกิดปัญหา
    (void)message;
    printf("ห้ามส่ง user input เป็น format string โดยตรง\n");
}

// ปลอดภัย: ใช้ %s เสมอ
void print_message_safe(const char *message) {
    if (message == NULL) {
        printf("(null message)\n");
        return;
    }
    printf("%s\n", message);  // ใช้ %s — ปลอดภัย
}

// ปลอดภัย: การ log ที่ถูกต้อง
void log_error(const char *filename, int line, const char *msg) {
    // format string เป็น literal เสมอ — ปลอดภัย
    fprintf(stderr, "[ERROR] %s:%d: %s\n", filename, line, msg);
}

// ปลอดภัย: การ build string จาก user input
void build_and_print(const char *username, int score) {
    if (username == NULL) return;
    
    char buffer[256];
    // ทุก field ถูกระบุ type ชัดเจน — ไม่มีทางที่ user input จะควบคุม format ได้
    int ret = snprintf(buffer, sizeof(buffer),
                       "User: %s | Score: %d", username, score);
    if (ret > 0 && (size_t)ret < sizeof(buffer)) {
        printf("%s\n", buffer);
    }
}

// ทดสอบว่าแม้ input มี format specifier ก็ปลอดภัย
int main(void) {
    const char *user_input = "Hello %s %n %x World";
    
    // ปลอดภัย: user_input ถูกปฏิบัติเป็น plain string
    print_message_safe(user_input);
    
    // ปลอดภัย: build string with format
    build_and_print("Alice%20s", 100);
    
    log_error(__FILE__, __LINE__, "example log message");
    
    return 0;
}
```

### กฎทอง: Format String Safety

```c
#include <stdio.h>
#include <syslog.h>

// สรุปกฎทอง
void format_string_rules_demo(const char *user_input) {
    if (user_input == NULL) return;
    
    // ผิด: อย่าทำ
    // printf(user_input);
    // fprintf(stderr, user_input);
    // syslog(LOG_INFO, user_input);
    // sprintf(buffer, user_input);
    
    // ถูก: ทำแบบนี้เสมอ
    printf("%s\n", user_input);              // printf: ใช้ %s
    fprintf(stderr, "%s\n", user_input);     // fprintf: ใช้ %s
    syslog(LOG_INFO, "%s", user_input);      // syslog: ใช้ %s
    
    char buffer[256];
    snprintf(buffer, sizeof(buffer), "%s", user_input);  // snprintf: ใช้ %s
    (void)buffer;
}

int main(void) {
    format_string_rules_demo("Test input with %s %n specifiers");
    return 0;
}
```

---

## Step 918: Compiler Security Flags

Compiler มี flags หลายตัวที่ช่วยป้องกัน security issues ตั้งแต่ compile time และ runtime

### รายการ Compiler Flags ที่สำคัญ

```bash
# สำหรับ C
gcc -Wall -Wextra -Wformat=2 -Wformat-overflow=2 \
    -fstack-protector-strong \
    -D_FORTIFY_SOURCE=2 \
    -pie -fPIE \
    -Wl,-z,relro,-z,now \
    -o program source.c

# สำหรับ C++
g++ -std=c++17 -Wall -Wextra -Wformat=2 -Wformat-overflow=2 \
    -fstack-protector-strong \
    -D_FORTIFY_SOURCE=2 \
    -pie -fPIE \
    -Wl,-z,relro,-z,now \
    -o program source.cpp
```

### อธิบาย Flag แต่ละตัว

| Flag | หน้าที่ |
|------|--------|
| `-Wall` | เปิด warning พื้นฐานทุกตัว |
| `-Wextra` | เปิด warning เพิ่มเติมที่ `-Wall` ไม่รวม |
| `-Wformat=2` | ตรวจ format string อย่างเข้มงวด |
| `-Wformat-overflow=2` | เตือนเมื่อ formatted output อาจ overflow |
| `-fstack-protector-strong` | เพิ่ม stack canary เพื่อตรวจ stack overflow |
| `-D_FORTIFY_SOURCE=2` | ใช้ safe version ของ string/memory functions |
| `-pie -fPIE` | สร้าง Position Independent Executable (ASLR support) |
| `-Wl,-z,relro` | ทำ GOT/PLT เป็น read-only หลัง linking |
| `-Wl,-z,now` | Resolve symbols ทั้งหมดตอน load (ไม่ lazy) |

### ตัวอย่างโค้ดที่แสดงผลของ Flags

```c
// demo_flags.c — compile ด้วย: gcc -Wall -Wextra -Wformat=2 -o demo demo_flags.c

#include <stdio.h>
#include <string.h>
#include <stdlib.h>

// -Wall จะเตือนเกี่ยวกับตัวแปรที่ไม่ได้ใช้
// -Wextra จะเตือนเกี่ยวกับ unused parameters
// -Wformat=2 จะเตือนถ้า format string ไม่ตรงกับ argument

void safe_function(void) {
    // _FORTIFY_SOURCE=2 ทำให้ string functions ตรวจ bounds ที่ runtime
    char dest[10];
    const char *src = "Hello";
    
    // snprintf ปลอดภัยอยู่แล้ว
    snprintf(dest, sizeof(dest), "%s", src);
    printf("result: %s\n", dest);
}

// ตัวอย่างการใช้ stack variable — -fstack-protector-strong ปกป้อง stack frame นี้
void function_with_buffer(void) {
    char local_buffer[64];
    
    // ใช้ fgets แทน scanf %s เพื่อความปลอดภัย
    // fgets(local_buffer, sizeof(local_buffer), stdin);
    
    // ในตัวอย่างนี้เราใช้ค่าคงที่
    snprintf(local_buffer, sizeof(local_buffer), "%s", "safe data");
    printf("local_buffer: %s\n", local_buffer);
}

int main(void) {
    safe_function();
    function_with_buffer();
    return 0;
}
```

### Makefile ที่มี Security Flags

```makefile
# Makefile สำหรับโปรเจกต์ที่ปลอดภัย

CC = gcc
CXX = g++
CFLAGS = -Wall -Wextra -Wformat=2 -Wformat-overflow=2 \
         -fstack-protector-strong \
         -D_FORTIFY_SOURCE=2 \
         -pie -fPIE \
         -std=c11

CXXFLAGS = -Wall -Wextra -Wformat=2 -Wformat-overflow=2 \
           -fstack-protector-strong \
           -D_FORTIFY_SOURCE=2 \
           -pie -fPIE \
           -std=c++17

LDFLAGS = -Wl,-z,relro,-z,now

# สำหรับ debug build: เพิ่ม sanitizers
DEBUG_FLAGS = -g -O1 \
              -fsanitize=address \
              -fsanitize=undefined \
              -fno-omit-frame-pointer

all: program

program: main.c
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $<

debug: main.c
	$(CC) $(CFLAGS) $(DEBUG_FLAGS) -o $@_debug $<

clean:
	rm -f program program_debug
```

---

## Step 919: AddressSanitizer และ Valgrind

### AddressSanitizer (ASan)

AddressSanitizer เป็น compiler instrumentation ที่ช่วยตรวจจับ memory bugs ที่ runtime โดยมี overhead ประมาณ 2x ซึ่งเหมาะสำหรับการ test และ development

ASan ตรวจจับ:
- Buffer overflow (stack, heap, global)
- Use-after-free
- Use-after-return
- Use-after-scope
- Double-free
- Memory leaks (ด้วย LeakSanitizer)

```c
// asan_demo.c
// compile ด้วย: gcc -g -O1 -fsanitize=address -fno-omit-frame-pointer -o asan_demo asan_demo.c
// รันด้วย: ./asan_demo
// ASan จะรายงาน bug พร้อม stack trace ที่ชัดเจน

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// ตัวอย่าง: heap buffer ที่ปลอดภัย
void safe_heap_usage(void) {
    int *arr = (int *)malloc(10 * sizeof(int));
    if (arr == NULL) {
        fprintf(stderr, "malloc failed\n");
        return;
    }
    
    // เขียนข้อมูลใน bounds เสมอ
    for (int i = 0; i < 10; i++) {
        arr[i] = i * i;
    }
    
    // อ่านข้อมูลใน bounds เสมอ
    for (int i = 0; i < 10; i++) {
        printf("arr[%d] = %d\n", i, arr[i]);
    }
    
    // free หลังใช้งาน
    free(arr);
    arr = NULL;  // ป้องกัน UAF
}

// ตัวอย่าง: stack buffer ที่ปลอดภัย
void safe_stack_usage(void) {
    char buffer[32];
    const char *msg = "Safe message";
    
    // ตรวจขนาดก่อน copy
    size_t msg_len = strlen(msg);
    if (msg_len < sizeof(buffer)) {
        memcpy(buffer, msg, msg_len + 1);
        printf("Buffer: %s\n", buffer);
    } else {
        printf("Message too long for buffer\n");
    }
}

int main(void) {
    printf("Running with AddressSanitizer enabled...\n");
    safe_heap_usage();
    safe_stack_usage();
    printf("All memory operations were safe!\n");
    return 0;
}
```

### UndefinedBehaviorSanitizer (UBSan)

```c
// ubsan_demo.c
// compile ด้วย: gcc -g -O1 -fsanitize=undefined -o ubsan_demo ubsan_demo.c

#include <stdio.h>
#include <limits.h>
#include <stdint.h>

// ตัวอย่างการหลีกเลี่ยง undefined behavior

// ปัญหา: signed integer overflow เป็น UB ใน C
// วิธีแก้: ตรวจก่อนคำนวณ
int32_t safe_signed_add(int32_t a, int32_t b) {
    if ((b > 0 && a > INT32_MAX - b) ||
        (b < 0 && a < INT32_MIN - b)) {
        // ถ้าจะ overflow ให้ return error value หรือ clamp
        return (b > 0) ? INT32_MAX : INT32_MIN;
    }
    return a + b;
}

// ปัญหา: left shift ของ negative number หรือ shift มากกว่า width เป็น UB
// วิธีแก้: ตรวจก่อน shift
uint32_t safe_left_shift(uint32_t value, unsigned shift) {
    if (shift >= 32) {
        return 0;
    }
    return value << shift;
}

// ปัญหา: dereference NULL pointer เป็น UB
// วิธีแก้: ตรวจ NULL เสมอ
int safe_dereference(const int *ptr) {
    if (ptr == NULL) {
        return 0;  // หรือ return error value
    }
    return *ptr;
}

int main(void) {
    printf("Safe arithmetic:\n");
    printf("INT32_MAX + 1 (safe): %d\n", safe_signed_add(INT32_MAX, 1));
    printf("100 + 200 (safe): %d\n", safe_signed_add(100, 200));
    
    printf("\nSafe shift:\n");
    printf("1 << 5 = %u\n", safe_left_shift(1, 5));
    printf("1 << 40 (would be UB) = %u\n", safe_left_shift(1, 40));
    
    printf("\nSafe dereference:\n");
    int value = 42;
    printf("non-null: %d\n", safe_dereference(&value));
    printf("null: %d\n", safe_dereference(NULL));
    
    return 0;
}
```

### Valgrind

Valgrind เป็นเครื่องมือ dynamic analysis ที่ไม่ต้องการ recompile แต่ช้ากว่า ASan ประมาณ 10-50x เหมาะสำหรับตรวจ memory leaks อย่างละเอียด

```bash
# compile ด้วย debug symbols
gcc -g -O0 -o myprogram myprogram.c

# ตรวจ memory errors
valgrind --tool=memcheck --error-exitcode=1 ./myprogram

# ตรวจ memory leaks อย่างละเอียด
valgrind --tool=memcheck \
         --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         --error-exitcode=1 \
         ./myprogram

# ดู call graph
valgrind --tool=callgrind ./myprogram
callgrind_annotate callgrind.out.*
```

### โปรแกรมที่ Valgrind จะผ่าน (Memory Clean)

```c
// valgrind_clean.c — โปรแกรมนี้ไม่มี memory leak
// ทดสอบด้วย: valgrind --leak-check=full ./valgrind_clean

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    char *name;
    int age;
} Person;

Person *create_person(const char *name, int age) {
    if (name == NULL) return NULL;
    
    Person *p = (Person *)malloc(sizeof(Person));
    if (p == NULL) return NULL;
    
    size_t name_len = strlen(name);
    p->name = (char *)malloc(name_len + 1);
    if (p->name == NULL) {
        free(p);
        return NULL;
    }
    
    memcpy(p->name, name, name_len + 1);
    p->age = age;
    return p;
}

void destroy_person(Person **pp) {
    if (pp == NULL || *pp == NULL) return;
    free((*pp)->name);
    (*pp)->name = NULL;
    free(*pp);
    *pp = NULL;
}

int main(void) {
    Person *alice = create_person("Alice", 30);
    Person *bob = create_person("Bob", 25);
    
    if (alice != NULL) {
        printf("Person: %s, age %d\n", alice->name, alice->age);
    }
    if (bob != NULL) {
        printf("Person: %s, age %d\n", bob->name, bob->age);
    }
    
    // ทำความสะอาดทุกอย่าง — Valgrind จะรายงาน "no leaks"
    destroy_person(&alice);
    destroy_person(&bob);
    
    printf("All memory freed. Valgrind should report no leaks.\n");
    return 0;
}
```

### เปรียบเทียบ ASan กับ Valgrind

| คุณสมบัติ | AddressSanitizer | Valgrind |
|----------|-----------------|---------|
| ต้อง recompile | ใช่ | ไม่ |
| Speed overhead | ~2x | ~10-50x |
| Stack overflow detection | ดีมาก | จำกัด |
| Heap overflow detection | ดีมาก | ดีมาก |
| Memory leak detection | ดี (ต้องเพิ่ม LSan) | ดีมาก |
| เหมาะสำหรับ | CI/CD, unit tests | Deep analysis |

---

## Step 920: Input Validation และ Secure Coding Guidelines

### หลักการ: Never Trust External Input

ข้อมูลทุกชนิดที่มาจากภายนอกโปรแกรม ไม่ว่าจะเป็น user input, network data, file contents, environment variables, command-line arguments หรือ configuration files ทั้งหมดต้องผ่านการ validate ก่อนใช้งาน

```c
// input_validation.c — ตัวอย่าง input validation ที่ครอบคลุม
// compile ด้วย: gcc -Wall -Wextra -fstack-protector-strong -o validation input_validation.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>
#include <limits.h>
#include <errno.h>

#define MAX_NAME_LEN    64
#define MAX_INPUT_LEN   256
#define MIN_AGE         0
#define MAX_AGE         150

// Validate และ sanitize ชื่อผู้ใช้
// คืน 0 ถ้าปลอดภัย, -1 ถ้าไม่ผ่าน
int validate_username(const char *input, char *clean_output, size_t out_size) {
    if (input == NULL || clean_output == NULL || out_size == 0) {
        return -1;
    }
    
    size_t len = strlen(input);
    
    // ตรวจความยาว
    if (len == 0) {
        fprintf(stderr, "Validation error: username cannot be empty\n");
        return -1;
    }
    if (len > MAX_NAME_LEN) {
        fprintf(stderr, "Validation error: username too long (%zu > %d)\n",
                len, MAX_NAME_LEN);
        return -1;
    }
    
    // ตรวจ characters — อนุญาตเฉพาะ alphanumeric และ underscore
    for (size_t i = 0; i < len; i++) {
        if (!isalnum((unsigned char)input[i]) && input[i] != '_') {
            fprintf(stderr,
                    "Validation error: invalid character '%c' at position %zu\n",
                    input[i], i);
            return -1;
        }
    }
    
    // ตัวแรกต้องเป็น letter
    if (!isalpha((unsigned char)input[0])) {
        fprintf(stderr, "Validation error: username must start with a letter\n");
        return -1;
    }
    
    // copy ข้อมูลที่ผ่านการตรวจแล้ว
    snprintf(clean_output, out_size, "%s", input);
    return 0;
}

// Parse integer อย่างปลอดภัย
int parse_integer(const char *str, int *result, int min_val, int max_val) {
    if (str == NULL || result == NULL) {
        return -1;
    }
    
    // ตรวจ string ว่าง
    if (*str == '\0') {
        fprintf(stderr, "Parse error: empty string\n");
        return -1;
    }
    
    // ใช้ strtol แทน atoi เพราะ strtol แจ้ง error ได้
    char *endptr = NULL;
    errno = 0;
    long value = strtol(str, &endptr, 10);
    
    // ตรวจ conversion errors
    if (errno == ERANGE) {
        fprintf(stderr, "Parse error: number out of long range\n");
        return -1;
    }
    if (endptr == str) {
        fprintf(stderr, "Parse error: no digits found in '%s'\n", str);
        return -1;
    }
    if (*endptr != '\0') {
        fprintf(stderr, "Parse error: trailing characters '%s'\n", endptr);
        return -1;
    }
    
    // ตรวจ range
    if (value < min_val || value > max_val) {
        fprintf(stderr,
                "Validation error: value %ld is out of range [%d, %d]\n",
                value, min_val, max_val);
        return -1;
    }
    
    *result = (int)value;
    return 0;
}

// อ่าน line จาก stdin อย่างปลอดภัย
// คืน จำนวน char ที่อ่าน, -1 ถ้า error
int read_line_safe(char *buffer, size_t buf_size) {
    if (buffer == NULL || buf_size == 0) {
        return -1;
    }
    
    if (fgets(buffer, (int)buf_size, stdin) == NULL) {
        return -1;
    }
    
    // ลบ newline ออก
    size_t len = strlen(buffer);
    if (len > 0 && buffer[len - 1] == '\n') {
        buffer[--len] = '\0';
    } else if (len == buf_size - 1) {
        // input ยาวเกินกว่า buffer — ล้าง stdin
        int c;
        while ((c = getchar()) != '\n' && c != EOF);
        fprintf(stderr, "Warning: input was truncated to %zu characters\n",
                buf_size - 1);
    }
    
    return (int)len;
}

// ตัวอย่าง: process user registration อย่างปลอดภัย
void process_registration(void) {
    char input_buf[MAX_INPUT_LEN];
    char clean_username[MAX_NAME_LEN + 1];
    int age;
    
    printf("=== User Registration (Secure Version) ===\n");
    
    // อ่านและ validate username
    printf("Enter username (letters, numbers, underscore only): ");
    if (read_line_safe(input_buf, sizeof(input_buf)) < 0) {
        fprintf(stderr, "Error reading username\n");
        return;
    }
    if (validate_username(input_buf, clean_username, sizeof(clean_username)) != 0) {
        printf("Registration failed: invalid username\n");
        return;
    }
    
    // อ่านและ validate age
    printf("Enter age: ");
    if (read_line_safe(input_buf, sizeof(input_buf)) < 0) {
        fprintf(stderr, "Error reading age\n");
        return;
    }
    if (parse_integer(input_buf, &age, MIN_AGE, MAX_AGE) != 0) {
        printf("Registration failed: invalid age\n");
        return;
    }
    
    // ใช้ข้อมูลที่ validate แล้วเท่านั้น
    printf("Registration successful!\n");
    printf("Username: %s\n", clean_username);
    printf("Age: %d\n", age);
}

int main(void) {
    process_registration();
    return 0;
}
```

### Validate Path Traversal

```c
// path_validation.c — ป้องกัน directory traversal
// compile ด้วย: gcc -Wall -Wextra -o path_demo path_validation.c

#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <limits.h>

// ตรวจว่า path ไม่มี directory traversal
// คืน 0 ถ้าปลอดภัย, -1 ถ้าไม่ปลอดภัย
int validate_filename(const char *filename) {
    if (filename == NULL) return -1;
    
    // ห้าม absolute path
    if (filename[0] == '/') {
        fprintf(stderr, "Security: absolute path rejected: %s\n", filename);
        return -1;
    }
    
    // ห้าม .. (directory traversal)
    if (strstr(filename, "..") != NULL) {
        fprintf(stderr, "Security: path traversal rejected: %s\n", filename);
        return -1;
    }
    
    // ห้าม null byte ใน string (สำหรับ C strings เรียบร้อยแล้ว
    // แต่ถ้า receive จาก network ต้องตรวจ embedded null)
    
    // อนุญาตเฉพาะ alphanumeric, dash, underscore, dot, slash
    for (size_t i = 0; filename[i] != '\0'; i++) {
        char c = filename[i];
        if (!((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') ||
              (c >= '0' && c <= '9') || c == '-' || c == '_' ||
              c == '.' || c == '/')) {
            fprintf(stderr, "Security: invalid character '%c' in filename\n", c);
            return -1;
        }
    }
    
    return 0;
}

// สร้าง path ที่ปลอดภัยภายใต้ base directory
int build_safe_path(const char *base_dir, const char *filename,
                    char *output, size_t out_size) {
    if (base_dir == NULL || filename == NULL ||
        output == NULL || out_size == 0) {
        return -1;
    }
    
    // validate filename ก่อน
    if (validate_filename(filename) != 0) {
        return -1;
    }
    
    // สร้าง path
    int written = snprintf(output, out_size, "%s/%s", base_dir, filename);
    if (written < 0 || (size_t)written >= out_size) {
        fprintf(stderr, "Error: path too long\n");
        return -1;
    }
    
    return 0;
}

int main(void) {
    char safe_path[PATH_MAX];
    const char *base = "/var/www/uploads";
    
    // ทดสอบ filename ต่างๆ
    const char *filenames[] = {
        "document.txt",       // ปลอดภัย
        "../../../etc/passwd", // อันตราย: traversal
        "/etc/shadow",        // อันตราย: absolute path
        "sub/dir/file.txt",   // ปลอดภัย (ถ้า sub/dir มีอยู่)
        "file with spaces",   // ไม่อนุญาต
        NULL
    };
    
    for (int i = 0; filenames[i] != NULL; i++) {
        printf("\nTesting: '%s'\n", filenames[i]);
        if (build_safe_path(base, filenames[i], safe_path, sizeof(safe_path)) == 0) {
            printf("  -> Safe path: %s\n", safe_path);
        } else {
            printf("  -> Rejected!\n");
        }
    }
    
    return 0;
}
```

### Secure Coding Guidelines: CERT C/C++ และ CWE

```cpp
// secure_guidelines_demo.cpp
// compile ด้วย: g++ -std=c++17 -Wall -Wextra -o guidelines secure_guidelines_demo.cpp

#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <stdexcept>
#include <regex>
#include <cassert>

// === CERT C++ Secure Coding Rules ===

// CERT Rule: INT30-C — ไม่ใช้ unsigned wrap around
// CERT Rule: INT32-C — ป้องกัน signed integer overflow
namespace cert_int {
    // CWE-190: Integer Overflow or Wraparound
    // ใช้ checked arithmetic
    int32_t add_checked(int32_t a, int32_t b) {
        if ((b > 0 && a > std::numeric_limits<int32_t>::max() - b) ||
            (b < 0 && a < std::numeric_limits<int32_t>::min() - b)) {
            throw std::overflow_error("Integer overflow in addition");
        }
        return a + b;
    }
}

// CERT Rule: MEM50-CPP — ไม่ access freed memory
// CERT Rule: MEM51-CPP — free memory อย่างถูกต้อง
namespace cert_mem {
    // ใช้ RAII และ smart pointers
    class SafeBuffer {
    public:
        explicit SafeBuffer(size_t size)
            : data_(std::make_unique<char[]>(size)), size_(size) {
            if (size == 0) {
                throw std::invalid_argument("Buffer size cannot be zero");
            }
        }
        
        char& at(size_t index) {
            if (index >= size_) {
                throw std::out_of_range("Buffer index out of range");
            }
            return data_[index];
        }
        
        const char& at(size_t index) const {
            if (index >= size_) {
                throw std::out_of_range("Buffer index out of range");
            }
            return data_[index];
        }
        
        size_t size() const { return size_; }
        
        // copy นั้นปลอดภัย — unique_ptr จัดการ memory
        // ไม่มีโอกาส UAF หรือ double-free
        
    private:
        std::unique_ptr<char[]> data_;
        size_t size_;
    };
}

// CERT Rule: STR50-CPP — ตรวจ string operations
namespace cert_str {
    // CWE-125: Out-of-bounds Read
    // CWE-787: Out-of-bounds Write
    
    // ใช้ std::string แทน C-style strings
    std::string sanitize_html(const std::string& input) {
        std::string output;
        output.reserve(input.size());
        
        for (char c : input) {
            switch (c) {
                case '<':  output += "&lt;";   break;
                case '>':  output += "&gt;";   break;
                case '&':  output += "&amp;";  break;
                case '"':  output += "&quot;"; break;
                case '\'': output += "&#x27;"; break;
                default:   output += c;        break;
            }
        }
        return output;
    }
    
    // Validate email ด้วย regex
    bool validate_email(const std::string& email) {
        // Pattern พื้นฐาน — production code ควรใช้ library ที่เชี่ยวชาญ
        static const std::regex email_pattern(
            R"([a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})"
        );
        return std::regex_match(email, email_pattern);
    }
}

// CERT Rule: ERR50-CPP — ไม่ละเว้น exceptions
// CERT Rule: ERR51-CPP — handle exceptions ที่เป็นไปได้ทั้งหมด
namespace cert_err {
    // Return type ที่บอก error ได้โดยไม่ต้องใช้ exception
    // (C++23 std::expected แต่นี่เป็น simple version)
    template<typename T>
    struct Result {
        bool ok;
        T value;
        std::string error_msg;
        
        static Result success(T val) {
            return {true, std::move(val), {}};
        }
        static Result failure(std::string msg) {
            return {false, T{}, std::move(msg)};
        }
    };
    
    Result<int> parse_port(const std::string& str) {
        try {
            size_t pos = 0;
            int port = std::stoi(str, &pos);
            if (pos != str.size()) {
                return Result<int>::failure("trailing characters in port number");
            }
            if (port < 1 || port > 65535) {
                return Result<int>::failure("port must be between 1 and 65535");
            }
            return Result<int>::success(port);
        } catch (const std::exception& e) {
            return Result<int>::failure(std::string("parse error: ") + e.what());
        }
    }
}

int main() {
    // ทดสอบ checked arithmetic
    std::cout << "=== Checked Arithmetic ===\n";
    try {
        int32_t result = cert_int::add_checked(100, 200);
        std::cout << "100 + 200 = " << result << "\n";
        cert_int::add_checked(std::numeric_limits<int32_t>::max(), 1);
    } catch (const std::overflow_error& e) {
        std::cout << "Overflow caught: " << e.what() << "\n";
    }
    
    // ทดสอบ safe buffer
    std::cout << "\n=== Safe Buffer ===\n";
    try {
        cert_mem::SafeBuffer buf(10);
        buf.at(0) = 'H';
        buf.at(1) = 'i';
        std::cout << "buf[0]=" << buf.at(0) << " buf[1]=" << buf.at(1) << "\n";
        buf.at(100);  // จะ throw
    } catch (const std::out_of_range& e) {
        std::cout << "Out of range caught: " << e.what() << "\n";
    }
    
    // ทดสอบ HTML sanitization
    std::cout << "\n=== HTML Sanitization ===\n";
    std::string dangerous = "<script>alert('XSS')</script>";
    std::string safe = cert_str::sanitize_html(dangerous);
    std::cout << "Original: " << dangerous << "\n";
    std::cout << "Sanitized: " << safe << "\n";
    
    // ทดสอบ email validation
    std::cout << "\n=== Email Validation ===\n";
    std::vector<std::string> emails = {
        "user@example.com",
        "invalid-email",
        "user@",
        "test.user+tag@subdomain.example.org"
    };
    for (const auto& email : emails) {
        std::cout << email << ": "
                  << (cert_str::validate_email(email) ? "valid" : "invalid")
                  << "\n";
    }
    
    // ทดสอบ port parsing
    std::cout << "\n=== Port Parsing ===\n";
    std::vector<std::string> ports = {"8080", "443", "99999", "abc", "80x"};
    for (const auto& p : ports) {
        auto result = cert_err::parse_port(p);
        if (result.ok) {
            std::cout << "Port '" << p << "' = " << result.value << "\n";
        } else {
            std::cout << "Port '" << p << "' invalid: " << result.error_msg << "\n";
        }
    }
    
    return 0;
}
```

### Static Analysis Tools

Static analysis ตรวจโค้ดโดยไม่ต้องรัน ช่วยหา bugs ตั้งแต่ก่อน compile

```bash
# ติดตั้ง tools
# Ubuntu/Debian:
# apt install cppcheck clang-tools

# Cppcheck: เครื่องมือ static analysis ฟรีสำหรับ C/C++
cppcheck --enable=all \
         --suppress=missingIncludeSystem \
         --error-exitcode=1 \
         --std=c11 \
         source.c

# สำหรับ C++ project
cppcheck --enable=all \
         --suppress=missingIncludeSystem \
         --error-exitcode=1 \
         --std=c++17 \
         src/

# clang-tidy: ตรวจตาม rules หลายชุด รวมถึง security
clang-tidy source.cpp \
    -checks='cert-*,cppcoreguidelines-*,bugprone-*,clang-analyzer-security-*' \
    -- -std=c++17

# clang static analyzer (scan-build)
scan-build --use-cc=gcc make

# PVS-Studio (commercial, มี free tier สำหรับ open source)
# pvs-studio-analyzer analyze -o pvs.log -e pvs.err
```

---

## สรุป Part 115: Security ใน C/C++

### ตารางสรุป Vulnerabilities และวิธีป้องกัน

| ช่องโหว่ | สาเหตุ | วิธีป้องกัน |
|---------|-------|-----------|
| Buffer Overflow | ไม่ตรวจ bounds | `snprintf`, ตรวจ length ก่อน |
| Use-After-Free | ใช้ pointer หลัง free | ตั้ง NULL หลัง free, ใช้ smart pointer |
| Double-Free | free สองครั้ง | ตรวจ NULL ก่อน free, ใช้ smart pointer |
| Integer Overflow | ไม่ตรวจ range | Checked arithmetic, `stdint.h` |
| Format String | ส่ง input เป็น format | ใช้ `%s` เสมอ, literal format string |
| NULL Dereference | ไม่ตรวจ NULL | ตรวจก่อน dereference เสมอ |

### Checklist สำหรับ Secure Code Review

```
[ ] ใช้ snprintf/strncpy แทน sprintf/strcpy
[ ] ตรวจ return value ของทุก malloc/calloc
[ ] ตั้ง pointer เป็น NULL หลัง free (C)
[ ] ใช้ smart pointer ใน C++
[ ] ตรวจ integer ก่อนคำนวณที่อาจ overflow
[ ] ใช้ format string literal เสมอ — ไม่ใช่ตัวแปร
[ ] Validate ทุก input จากภายนอก
[ ] ตรวจ bounds ก่อน array access ทุกครั้ง
[ ] Compile ด้วย security flags (-fstack-protector, etc.)
[ ] รัน ASan/Valgrind ก่อน release
[ ] ทำ static analysis ด้วย cppcheck หรือ clang-tidy
[ ] ตรวจ path traversal สำหรับ file operations
```

### การนำไปใช้ใน CI/CD Pipeline

```yaml
# .github/workflows/security-check.yml (ตัวอย่าง)
# ไม่ใช่ code จริง แต่แสดง concept

# steps ใน CI pipeline:
# 1. Build ด้วย security flags + ASan
#    gcc -Wall -Wextra -fsanitize=address -fstack-protector-strong
#
# 2. Run tests กับ sanitizer
#    ASAN_OPTIONS=halt_on_error=1 ./run_tests
#
# 3. Static analysis
#    cppcheck --error-exitcode=1 src/
#    clang-tidy src/*.cpp -- -std=c++17
#
# 4. Build production ด้วย hardening flags
#    gcc -O2 -fstack-protector-strong -D_FORTIFY_SOURCE=2 -pie -fPIE
```

### ขั้นตอนสำคัญที่ต้องจำ

**Step 913** — C/C++ ไม่มี automatic safety ต้องป้องกันเอง  
**Step 914** — Buffer Overflow: ใช้ `snprintf` และตรวจ bounds เสมอ  
**Step 915** — UAF/Double-Free: NULL หลัง free, ใช้ smart pointer  
**Step 916** — Integer Overflow: ตรวจ range ก่อนคำนวณ  
**Step 917** — Format String: ใช้ format literal เสมอ (`"%s"`)  
**Step 918** — Compiler Flags: `-fstack-protector-strong`, `-D_FORTIFY_SOURCE=2`, `-pie`  
**Step 919** — ASan + Valgrind: ใช้ตรวจ memory bugs ก่อน ship  
**Step 920** — Input Validation: sanitize ทุก external input, ใช้ static analysis

---

## แหล่งข้อมูลเพิ่มเติม

- **CERT C Coding Standard**: [wiki.sei.cmu.edu/confluence/display/c](https://wiki.sei.cmu.edu/confluence/display/c)
- **CERT C++ Coding Standard**: [wiki.sei.cmu.edu/confluence/display/cplusplus](https://wiki.sei.cmu.edu/confluence/display/cplusplus)
- **CWE Top 25 Most Dangerous Software Weaknesses**: [cwe.mitre.org/top25](https://cwe.mitre.org/top25)
- **AddressSanitizer**: [github.com/google/sanitizers](https://github.com/google/sanitizers)
- **Valgrind**: [valgrind.org](https://valgrind.org)
- **Cppcheck**: [cppcheck.sourceforge.io](http://cppcheck.sourceforge.io)
- **clang-tidy**: [clang.llvm.org/extra/clang-tidy](https://clang.llvm.org/extra/clang-tidy)

---

> **Part ถัดไป**: [Part 116 — Cross-Platform Development](./part-116-cross-platform.md)  
> เรียนรู้วิธีเขียนโค้ด C/C++ ที่ทำงานได้บน Windows, Linux และ macOS โดยใช้ CMake, conditional compilation และ platform abstraction layers
